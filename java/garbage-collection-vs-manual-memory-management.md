# Why Java Needs Garbage Collection but C++ Doesn't

See also: [java-memory-management.md](java-memory-management.md) for *where* things actually live in the JVM — stack vs. heap for local variables, the method area/metaspace, and the string pool — as opposed to this file's focus on *why* reclamation works the way it does.

## The core question

Both languages need memory to be reclaimed once it's no longer needed — otherwise every running program would eventually exhaust RAM. The difference is *who* does the reclaiming and *when*: C++ ties reclamation to a scope ending, decided at compile time; Java ties it to reachability, decided at runtime by a background collector. C++ *could* add a tracing garbage collector (nothing in the language forbids it, and libraries like the Boehm GC exist) — it's a deliberate design choice, not a technical impossibility either way.

## How C++ manages memory

- **Stack allocation, automatic destruction**: a local variable's destructor runs the instant it goes out of scope — deterministic, compiler-inserted, no runtime bookkeeping needed.
- **Heap allocation, two styles**:
  - Raw `new`/`delete` — fully manual. Forget the `delete` and the memory leaks forever; `delete` it twice, or use it after deleting, and behavior is undefined (a classic source of crashes and security vulnerabilities).
  - Smart pointers (`unique_ptr`, `shared_ptr`, `weak_ptr`, standard since C++11) — RAII (Resource Acquisition Is Initialization) applied to heap memory: the smart pointer's own destructor calls `delete` for you the moment it goes out of scope (`unique_ptr`) or the last owner disappears (`shared_ptr`'s reference count hits zero). This is automatic and deterministic, but still not a garbage collector — it's scope/ownership-based, not reachability-based.

```cpp
void process() {
    auto conn = std::make_unique<Connection>(); // heap-allocated
    conn->send(data);
} // conn's destructor runs here, deterministically, closing the connection — no GC involved
```

## How Java manages memory

- Every non-primitive value is heap-allocated (the JIT can sometimes prove an object never escapes a method and stack-allocate it under the hood, but that's an optimization, not something the language exposes).
- There is no `delete` keyword — the language deliberately gives you no way to manually free an object.
- Instead, the JVM's garbage collector periodically walks from a set of **GC roots** (local variables on every thread's stack, static fields, JNI references) and marks every object reachable from them. Anything left unmarked is garbage — nothing in the program can possibly reach it anymore — and its memory is reclaimed.
- Modern collectors (G1, ZGC, Shenandoah) do this incrementally/concurrently with the application, achieving sub-millisecond pause times — the old "GC means unpredictable multi-second freezes" reputation is largely outdated.

```java
void process() {
    Connection conn = new Connection(); // heap-allocated
    conn.send(data);
} // conn becomes unreachable here, but its memory is reclaimed whenever the GC next runs — not necessarily immediately
```

## Why the difference — design philosophy, not capability

- **Zero-overhead principle**: C++'s guiding rule is "don't pay for what you don't use." A tracing GC requires runtime bookkeeping on *every* object (metadata, write barriers to track reference changes) even for programs that never need it — unacceptable for C++'s target domains (OS kernels, embedded firmware, game engines, real-time audio) where that overhead and pause unpredictability are the whole problem being avoided.
- **Safety and simplicity over control**: Java's designers accepted the overhead of a managed runtime specifically to eliminate an entire class of bugs — dangling pointers, use-after-free, double-free — that plagued C/C++ codebases, in exchange for giving up precise control over *when* memory is freed.
- **Reference cycles**: a tracing GC handles cycles for free — if object A references B and B references A, but nothing external reaches either, both get collected, since reachability (not reference count) is what matters. C++'s `shared_ptr` uses reference counting, which does **not** handle cycles automatically — A and B keep each other's count above zero forever unless one side deliberately uses `weak_ptr` to break the cycle.
- **Non-memory resources**: neither approach lets a language ignore file handles, sockets, or locks — Java's GC timing is intentionally unreliable for this (finalizers are deprecated), so Java bolts on `try-with-resources`/`AutoCloseable` as an opt-in, scope-based mechanism — effectively borrowing C++'s RAII idea for the one category of resource where GC's "eventually, whenever" timing genuinely isn't good enough.

## Deciding between GC and manual/RAII memory management

| Question | Real-life example (GC) | Real-life example (Manual/RAII) |
|---|---|---|
| Do you need deterministic, sub-millisecond-predictable resource release? | A typical e-commerce backend in Java, where an occasional few-millisecond GC pause is invisible against a 200ms average request — not worth the added complexity of manual lifetime tracking | Pacemaker or flight-control firmware in C++, where a GC pause mid-loop could miss a hard real-time deadline — RAII frees resources the instant a scope ends, with zero surprise pauses |
| Is avoiding whole bug classes (use-after-free, double-free) worth more than fine-grained control? | A large enterprise Java codebase with hundreds of contributors, where use-after-free bugs across that many hands would be a security nightmare — GC removes the bug class by construction | A small, tightly-reviewed C++ game engine core, where a handful of senior engineers want exact control over allocation patterns for performance profiling |
| Can the target environment afford a managed runtime's memory/CPU overhead? | A cloud service on commodity servers with gigabytes of RAM, where the JVM's overhead is a rounding error | Firmware on a microcontroller with kilobytes of RAM (an IoT sensor), with no room for a JVM or GC metadata at all |
| Do your data structures form reference cycles that need reclaiming automatically? | A Java UI framework where widgets hold parent/child back-references forming cycles — the tracing collector reclaims them once nothing external points in, no extra code needed | A C++ graph using `shared_ptr` for a similar parent/child structure — must deliberately use `weak_ptr` for the back-reference, or the cycle leaks forever, silently |
| Do you need precise, compiler-enforced cleanup for non-memory resources (files, sockets, locks)? | Java can't rely on GC timing for this — needs `try-with-resources`/`AutoCloseable`, an opt-in mini-RAII bolted on for exactly this reason | A C++ `std::lock_guard`/`std::ifstream` — the destructor releases the lock/closes the file the instant the variable's scope ends, no exceptions |

## Common memory leak patterns in Java

A tracing GC only reclaims what's *unreachable*. Every one of these "leaks" is really the same root cause — something still holds a reference to an object long after the program is logically done with it — the GC is doing its job correctly; the bug is in what the program kept referencing.

- **Static collections that only grow**: a `static` `List`/`Map` used as an ad-hoc cache, with entries added but never removed. The static field is a permanent GC root, so every entry put in stays reachable — and unreclaimed — forever.
  ```java
  static final Map<String, Report> cache = new HashMap<>(); // nothing ever calls .remove() — grows without bound
  ```
  Fix: a bounded/evicting cache (an LRU `LinkedHashMap`, Caffeine, etc.) or a `WeakHashMap` if entries should disappear once nothing else references the key.
- **Listeners/callbacks registered but never unregistered**: a long-lived publisher (an event bus, a Swing/Android component, a manually-managed `ApplicationListener` list) holds a list of subscribers. A short-lived object registers itself and is never removed — the long-lived, reachable publisher now keeps that short-lived object (and everything it references) alive indefinitely, well past its intended lifetime.
- **`ThreadLocal` values never `.remove()`d, especially inside a thread pool**: pooled threads (in an `ExecutorService`, a servlet container) are reused indefinitely rather than discarded per task. A `ThreadLocal` value set during one task but never explicitly removed stays attached to that thread object for the thread's entire lifetime — effectively forever, since the pool never lets the thread die. This is exactly the "ambient context doesn't propagate automatically, and doesn't clean up automatically either" problem noted in [spring-execution-contexts-and-hooks.md](../spring-boot/spring-execution-contexts-and-hooks.md) — the fix there is the same fix here: explicit `.remove()` in a `finally` block once the task completes.
- **Non-static inner classes holding an implicit outer-instance reference**: a non-`static` inner class instance always implicitly holds a reference to its enclosing instance. Hand that inner instance off somewhere long-lived (a callback, a separate thread) and the outer instance — and everything *it* references — leaks too, even though nothing external actually needed it anymore.
  ```java
  class ReportGenerator {
      byte[] hugeBuffer = new byte[100_000_000];
      class Callback { void onDone() { /* ... */ } } // implicitly holds `ReportGenerator.this`
  }
  // handing a Callback instance to a long-lived registry keeps the 100MB buffer alive too
  ```
  Fix: make the inner class `static` (it then holds no implicit outer reference) unless it genuinely needs the enclosing instance.
- **ClassLoader leaks** (application servers, hot redeploy): if anything outside a webapp's own classloader ends up referencing an object that classloader created — a JDBC driver left registered in the shared `DriverManager`, a `ThreadLocal` on a shared pool thread, a static field in a JDK class — the *entire classloader*, and every class and object it ever loaded, becomes unreclaimable even after the app is undeployed. Repeated redeploys accumulate these until `OutOfMemoryError: Metaspace`. See [java-program-execution.md](java-program-execution.md)'s "How a ClassLoader actually works" for why a classloader can hold this much reachable state in the first place.
- **(Historical) `String.substring()` before Java 7**: pre-Java 7, a substring shared the original, full backing character array rather than copying — so holding a tiny substring of a huge string kept the *entire* huge array alive for as long as that substring reference existed. Fixed in Java 7+, where `substring()` copies; worth knowing if you ever encounter code written for older JVMs.

## Real-life analogy

C++ with RAII is like a hotel where your checkout time is written into the booking itself — the moment your stay (scope) ends, housekeeping (the destructor) is called immediately and deterministically, every time, with no ambiguity about when the room gets cleaned. Raw `new`/`delete` is like doing your own laundry and having to remember to switch the machine off yourself — forget, and it runs forever wasting resources (a leak); switch off someone else's mid-cycle and you've broken their load (a dangling pointer/double free).

Java is like living in a large apartment building with a cleaning service (the garbage collector) that periodically walks every room, checking "does anyone still hold a key to this room (is it reachable)?" If no one does, it's cleaned out and reclaimed for a new tenant — you never have to remember to clean up yourself, and even two roommates who only have keys to each other's rooms (a reference cycle) still get evicted once no one *outside* that pair holds a key either. The trade-off: you don't get to choose exactly when the cleaning crew shows up.

## Common gotchas

- **"No `delete`" doesn't mean Java can't leak memory** — see "Common memory leak patterns in Java" above. Java's actual equivalent of a leak: not forgetting to free, but forgetting to stop referencing.
- **Modern C++ rarely uses raw `new`/`delete` directly** — idiomatic C++11+ code reaches for smart pointers almost everywhere, making manual memory bugs far less common in practice than the language's reputation suggests.
- **`shared_ptr` cycles leak silently** — unlike Java's tracing GC, reference counting has no way to notice "this cluster of objects is only reachable from itself"; breaking cycles with `weak_ptr` is the programmer's responsibility.
- **GC pause times are not what they used to be** — G1, ZGC, and Shenandoah target sub-millisecond pauses even on multi-gigabyte heaps; "Java has unpredictable stop-the-world pauses" is outdated for most current workloads.
