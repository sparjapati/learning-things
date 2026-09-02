# Java Memory Management: Stack, Heap, Method Area, and String Pool

> "Premature optimization is the root of all evil."
> — Donald Knuth
>
> *Worth remembering before tuning any of the regions below — but you can't tune what you can't name.*

See also: [java-concurrency.md](java-concurrency.md) for why per-thread stacks vs the shared heap make visibility a problem, and what `synchronized`, `volatile`, and the rest do about it.

See also: [garbage-collection-vs-manual-memory-management.md](garbage-collection-vs-manual-memory-management.md) for *why* Java needs a collector at all (vs. C++'s RAII), the GC-roots/reachability mechanism, and a decision checklist for GC vs. manual memory management. This file is about *where things physically live* — the JVM's runtime memory layout — rather than how it gets reclaimed.

Java memory isn't just "stack vs heap." The JVM spec defines several distinct runtime data areas, and knowing what lives in each one explains a lot of otherwise-confusing behavior: why passing an object only ever copies a reference, why unbounded recursion and a huge object graph fail with two *different* errors, and why `==` on strings sometimes "just works" and sometimes doesn't.

## The JVM's runtime data areas

**Per-thread** (every thread gets its own, private copy):

| Area | Holds |
|---|---|
| PC (Program Counter) Register | The address of the JVM instruction this thread is currently executing |
| JVM Stack | A stack of *frames*, one per in-progress method call on this thread — each frame holds that method's local variables array and operand stack |
| Native Method Stack | The equivalent of the JVM Stack, but for native (non-Java, e.g. JNI) method calls |

**Shared across the whole JVM process**:

| Area | Holds |
|---|---|
| Heap | Every object and array ever created with `new` — garbage collected, subdivided into generations (below) |
| Method Area / Metaspace | Per-class data: the runtime constant pool, field/method metadata, method bytecode, static variables |
| String Pool (String Intern Pool) | A deduplicated cache of `String` objects — lives inside the regular heap since Java 7 |

## Where do method-local variables actually live?

- **Local primitives** (`int`, `boolean`, `double`, …): stored directly, by value, in the current stack frame's local variable array. When the method returns, the frame — and that value with it — is popped instantly. No GC involvement, ever.
- **Local references** (`String s`, `Order o`, any object type): the *reference itself* (a handle, conceptually a pointer) lives in the stack frame, same as a primitive. The *object it points to* is allocated on the heap via `new`. So `Order o = new Order();` puts a small reference on the stack, while the actual `Order` and its fields live on the heap.
- When the method returns, the stack frame (and its reference variable) disappears immediately — but the heap object it pointed to is untouched. It keeps existing until the GC determines nothing reachable points to it anymore, which could be the very next collection or much later.
- **Instance fields** (fields belonging to an object) always live on the heap as part of that object's own memory, whether they're primitives or references — an `int` field inside an `Order` lives inside that `Order`'s heap block, not on any stack.
- **Static fields** live in the Method Area/Metaspace, as part of the class's own metadata — not on the heap, not on any stack — for as long as that class stays loaded.

```java
void placeOrder() {
    int quantity = 3;          // primitive: value lives directly in this stack frame
    Order order = new Order(); // reference "order" lives in this stack frame...
    order.setQuantity(quantity); // ...but the Order object itself lives on the heap
} // frame is popped here; `quantity` and the `order` reference vanish instantly.
  // The Order object on the heap survives until GC proves nothing reaches it.
```

**Escape analysis** (a JIT optimization, not a language guarantee): if the compiler can prove an object never "escapes" a method — never returned, stored in a field, or passed elsewhere — it may allocate it on the stack instead, or eliminate the allocation entirely (scalar replacement), skipping the heap and GC for it completely. This is invisible and non-guaranteed; you can't rely on or force it.

## The heap in detail — generational structure

Most collectors split the heap based on the "weak generational hypothesis" — most objects die young, so that region is worth collecting far more often and cheaply than the whole heap:

- **Young Generation**: `Eden` (where new objects are first allocated) plus two `Survivor` spaces (`S0`/`S1`). A **minor GC** collects Eden; anything still reachable is copied into a Survivor space, and after surviving enough minor GCs (the tenuring threshold), gets promoted into the Old Generation.
- **Old Generation (Tenured)**: long-lived objects. Collected less often via a **major/full GC**, typically more expensive since the region is larger.
- **(Historical) Permanent Generation (PermGen)**: pre-Java 8, held class metadata and the string pool, inside the heap with a fixed size — a frequent source of `OutOfMemoryError: PermGen space`. Removed in Java 8, replaced by Metaspace, which lives in native (off-heap) memory and grows automatically.

## Compressed oops: why a 64-bit JVM stores 32-bit references

An **oop** is HotSpot's name for an *ordinary object pointer* — a reference to a heap object. Every reference field, every element of a reference array, and the class pointer inside each object header is an oop.

On a 64-bit JVM a native oop is **8 bytes**. That was a real problem when the industry moved from 32-bit: the same program suddenly needed roughly 1.5× the heap for pointer-heavy structures, and each cache line held half as many references — the so-called "64-bit tax," paid in memory and in cache misses.

**Compressed oops** (`-XX:+UseCompressedOops`) fix it by storing references as **32-bit scaled offsets from a heap base** instead of absolute addresses. The trick rests on alignment: Java objects are 8-byte aligned, so the low 3 bits of every object address are always zero and carry no information. Drop them:

```
encode:  narrow = (address - heapBase) >>> 3
decode:  address = heapBase + (narrow << 3)

reach:   2^32 slots × 8 bytes = 32 GB of heap addressable with 32-bit references
```

HotSpot picks the cheapest of three modes automatically: **unscaled** (heap in the low 4 GB — the narrow value *is* the address), **zero-based** (heap mapped low enough that `heapBase` is 0, so decoding is a single shift with no add), or **base + shift** for anything higher. The shift usually folds into the CPU's addressing mode, so the runtime cost is close to nothing — and the smaller footprint often makes the program measurably *faster* through better cache utilization.

### The 32 GB cliff

Compressed oops are **on by default whenever the max heap fits under the ~32 GB limit**, and silently **off** above it. That produces the most counter-intuitive tuning result in the JVM:

```
-Xmx31g   → compressed oops ON  → every reference 4 bytes
-Xmx33g   → compressed oops OFF → every reference 8 bytes
```

A 33 GB heap can hold **less live data** than a 31 GB one, because every reference in the entire heap just doubled. The practical rule: stay under ~32 GB, or go far enough above it (roughly 48 GB+) that the extra raw capacity outweighs the loss. `-XX:ObjectAlignmentInBytes=16` raises the ceiling to 64 GB by shifting 4 bits instead of 3, at the cost of more per-object padding.

### What it changes about object size

| | Compressed oops on | off |
|---|---|---|
| Mark word | 8 bytes | 8 bytes |
| Klass pointer | 4 bytes (with `UseCompressedClassPointers`) | 8 bytes |
| Each reference field / reference array element | 4 bytes | 8 bytes |
| Typical header | 12 bytes (padded to 16) | 16 bytes |

Compressed **class** pointers are a separate but related switch (`-XX:+UseCompressedClassPointers`), which is why the JVM has a dedicated *Compressed Class Space* (1 GB by default, `-XX:CompressedClassSpaceSize`) in metaspace.

### Two nuances worth knowing

- **ZGC doesn't use compressed oops at all** — its coloured pointers need the full 64 bits, so choosing ZGC means accepting 8-byte references regardless of heap size.
- **"The address" of a Java object is an abstraction anyway.** With compressed oops a reference isn't a machine address but an encoded offset — which is one more reason references can't be persisted or handed to native code (see [garbage-collection-vs-manual-memory-management.md](garbage-collection-vs-manual-memory-management.md) on relocation, and [java-serialization.md](java-serialization.md) on why serialization exists).

To see which mode you're in: `java -Xlog:gc+heap+coops -version` prints the heap address range and the compressed-oops mode; the JOL (Java Object Layout) tool shows the resulting per-object field layout and padding.

## String Pool in detail

- A string **literal** written directly in source (`"hello"`) is automatically interned — placed in the pool if not already present, and every occurrence of that literal anywhere in the program reuses the *same* `String` object.
- `new String("hello")` bypasses the pool entirely — it explicitly allocates a brand-new `String` object on the regular heap, even though `"hello"` already exists in the pool.
- `.intern()` manually adds a heap-created string's content to the pool (or returns the existing pooled instance if it's already there) — useful for deduplicating many separately-created-but-equal strings.
- Since Java 7, the pool lives in the regular heap rather than PermGen, which made pooled strings properly garbage-collectible like anything else, instead of only clearing on a full PermGen GC.
- This is safe *because* `String` is immutable — many references can share one pooled object with zero risk that mutating it through one reference corrupts it for every other holder.

```java
String a = "hello";
String b = "hello";
String c = new String("hello");

a == b;          // true  — both point to the same pooled literal
a == c;          // false — c is a distinct object on the heap
a.equals(c);      // true  — same content, which is what actually matters almost always
a == c.intern();  // true  — intern() returns the pooled instance
```

## Real-life analogy

Think of a large corporate office building. The **stack** is each employee's own desk drawer — private, holding only their current task's notes (local variables); when they finish that task and move to the next (a method returns), the drawer's contents for that task are cleared instantly, no cleanup crew involved. The **heap** is the shared warehouse where actual products (objects) sit — anyone holding a requisition slip (a reference) can reach a product, and the warehouse doesn't remove one just because one particular drawer note pointing to it got cleared; only once *no one anywhere* in the building holds a slip to it does the cleanup crew (GC) actually take it away. The **Method Area/Metaspace** is the company's one shared policy binder — a single copy of "how does an `Order` work" (class structure, static data), never duplicated per drawer or per warehouse shelf. The **String Pool** is a shared bulletin board of common stock phrases ("Please hold") — instead of every employee handwriting their own copy of a common phrase (allocating a new `String`), they all just point to the one posted note; anyone who insists on their own private handwritten copy anyway (`new String(...)`) can have one, it just costs extra paper for no shared benefit.

## Common gotchas

- **"Objects live on the stack" is a common misconception** — only their references do. The object itself is always heap-allocated, barring invisible JIT escape-analysis optimizations you can't rely on or control.
- **`StackOverflowError` vs `OutOfMemoryError`**: unbounded/too-deep recursion exhausts a thread's stack → `StackOverflowError`. Too many long-lived, still-reachable objects exhausts the heap → `OutOfMemoryError: Java heap space`. Generating huge numbers of classes at runtime exhausts Metaspace → `OutOfMemoryError: Metaspace`. Three different resources, three different failures.
- **`==` on strings "works by accident" for literals** — two literals are `==` only because they share one pooled object. That breaks the instant either side is built via `new String(...)` or runtime concatenation (e.g. `s1 + s2`, which allocates a fresh `String`), which is why `.equals()`, not `==`, is the correct comparison for content.
- **Metaspace isn't fixed-size by default, unlike PermGen** — it grows automatically into native memory (bounded only if `-XX:MaxMetaspaceSize` is set), which fixed the classic `PermGen space` OOM from apps that loaded many classes (e.g. hot-redeploying app servers) — but an unbounded classloader leak can still exhaust *system* memory instead.
