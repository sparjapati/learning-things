# Java Memory Management: Stack, Heap, Method Area, and String Pool

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
