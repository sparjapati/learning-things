# What Happens When You Run `java Demo`

See also: [java-memory-management.md](java-memory-management.md) for the runtime data areas (heap, method area/metaspace, stack) that get set up during the startup this file describes, and [garbage-collection-vs-manual-memory-management.md](garbage-collection-vs-manual-memory-management.md) for what the GC threads spun up during startup actually do afterward.

`javac Demo.java` and `java Demo` are two entirely separate steps, done by two entirely separate programs. `javac` (compile time) translates your source into portable bytecode once; `java` (run time) is what actually turns that bytecode into a running program, every time you launch it. This is that second step, in order.

## 1. The launcher starts a JVM process

`java` is a small native launcher program (part of the JDK), not the JVM itself. It creates an actual JVM instance in-process (loading `libjvm.so`/`jvm.dll` and calling `JNI_CreateJavaVM` under the hood), passing along the classpath, any `-X`/`-XX` options, the main class name (`Demo`), and the program arguments. The JVM process then initializes its core runtime structures — the heap, method area/metaspace, thread management (see [java-memory-management.md](java-memory-management.md)) — and spins up several background threads besides your program's own: GC threads, JIT compiler threads, a reference-handler thread, a signal-dispatcher thread.

It looks for `Demo.class` on the classpath — compiled bytecode, not `Demo.java` — unless you're using the newer single-file source launch (JEP 330, Java 11+), where `java Demo.java` compiles it in memory and runs it in one step, intended for quick scripts rather than production use.

## 2. Class loading

Getting `Demo.class`'s bytecode into the JVM is handled by a hierarchy of classloaders, each delegating to its parent first:

| Classloader | Loads |
|---|---|
| Bootstrap | Core JDK classes (`java.lang.*`, `java.util.*`) — native code, part of the JVM itself |
| Platform (Java 9+; formerly "Extension") | Other platform modules, not part of core `java.base` |
| Application / System | Your classes — `Demo.class`, from the classpath (`-cp` or the current directory by default) |

**Delegation model**: asked to load a class, a classloader first asks its parent; only if the parent can't find it does the child try itself. This is what stops your own code from silently shadowing a core class like `java.lang.String`.

Classes are loaded **lazily** — on first active use, not all upfront. A class `Demo` references but never actually touches at runtime (no instance created, no static member accessed) may never get loaded at all. This is why a missing, unused dependency's `ClassNotFoundException` sometimes only shows up the first time a specific code path actually runs, not at startup.

### How a ClassLoader actually works

A `ClassLoader` isn't a JVM-internal black box — `java.lang.ClassLoader` is an ordinary (if special-cased) Java class, and the loaders in the table above are real instances of it (Bootstrap is the exception, represented as `null` in Java code since it predates being expressible as a Java object). Three methods matter:

- **`loadClass(name)`** — the public entry point, implementing the delegation algorithm: check this loader's own cache of already-loaded classes → if not cached, delegate to the parent's `loadClass()` → only if the parent can't find it, call this loader's own `findClass()`.
- **`findClass(name)`** — where a loader actually *locates* the bytecode it's responsible for (reading a `.class` file off disk, fetching bytes over a network, decrypting a custom format, generating bytecode on the fly). The default implementation just throws `ClassNotFoundException`; this is the method a custom loader overrides.
- **`defineClass(name, bytes, ...)`** — a `final` method that hands raw bytes to the JVM, which verifies and parses them into an actual `Class` object. Every loader eventually calls this once it has bytes, whichever way it got them.

```java
class NetworkClassLoader extends ClassLoader {
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] bytes = fetchOverNetwork(name); // a custom bytecode source
        return defineClass(name, bytes, 0, bytes.length);
    }
}
```

Custom classloaders like this are how plugin systems (load a plugin's classes from a JAR discovered at runtime, never on the original classpath), hot-reloading dev tools, and application servers (isolating each deployed app's classes from every other app's) all work — none of it needs JVM support beyond what `ClassLoader` already exposes.

### A class's real identity is (name, loader) — not just its name

The JVM treats two classes as *the same type* only if they share a fully-qualified name **and** were loaded by the same classloader instance. Load identical bytecode for `com.example.Widget` through two different classloaders and you get two distinct, mutually incompatible `Class` objects — an instance made by one can't be cast to the other, even though the source was byte-for-byte identical. This is what lets multi-app containers (app servers, OSGi, plugin frameworks) run two different versions of the same library side by side without clashing — and it's also the classic source of a confusing `ClassCastException: Widget cannot be cast to Widget` in exactly those environments.

### Parent-first vs. child-first (breaking the default delegation)

The default `loadClass()` algorithm always asks the parent first, deliberately, so your own code can never accidentally shadow a core class like `java.lang.String`. Some environments explicitly invert this to **child-first** (check this loader's own classes before asking the parent): a servlet container like Tomcat gives each deployed web app its own classloader that checks the app's own `WEB-INF/lib` *before* delegating to the shared container classloader, so two different apps — or an app and the container itself — can each depend on a different version of the same library without conflict. OSGi goes further still, letting each module declare exactly which packages it exports/imports rather than an all-or-nothing parent-first search.

A classloader is also a GC root of sorts by extension — as long as anything anywhere holds a reference to any object (or class) it loaded, the whole classloader and everything it ever loaded stays reachable; see "ClassLoader leaks" in [garbage-collection-vs-manual-memory-management.md](garbage-collection-vs-manual-memory-management.md) for how that becomes a real production problem in exactly the app-server/hot-redeploy scenario described above.

### What exactly is "the classpath"?

The classpath is the ordered list of locations — directories and JAR/ZIP files — the Application classloader searches to find a `.class` file by name. It's assembled from one of these, depending on how you launched:

- **`-classpath`/`-cp` flag** (or the `CLASSPATH` environment variable, if given no flag at all): an explicit, ordered list — colon-separated on Unix/macOS, semicolon-separated on Windows, e.g. `java -cp .:lib/commons.jar:lib/gson.jar Demo`.
- **Current directory, by default**: with no `-cp` at all, the JVM searches just `.`.
- **A JAR's manifest `Class-Path` entry**: `java -jar app.jar` ignores any `-cp` flag entirely — the classpath instead comes from that JAR's `META-INF/MANIFEST.MF` `Class-Path:` line.

What counts as one entry:
- A **directory** entry must be the *root* of the package hierarchy — if `Demo.class` is really at `com/example/Demo.class` on disk, the entry is the directory containing `com/`, not `com/example/` itself, since the classloader rebuilds the path from the fully-qualified class name (`com.example.Demo` → `com/example/Demo.class`).
- A **JAR** entry is searched as if its contents were unpacked at that location.
- A trailing **`/*`** (e.g. `lib/*`) expands to every `.jar` in that directory.

Entries are searched left to right within this one classpath — but that's a separate concern from the classloader delegation hierarchy above; the Application classloader as a whole is still asked *last*, after Bootstrap and Platform get their chance.

In practice you rarely hand-build a classpath: Maven/Gradle compute it from your declared dependencies automatically, and Spring Boot's executable "fat jar" instead uses a custom classloader that reads dependency JARs nested *inside* the one jar, bypassing a flat classpath list entirely. Java 9's module system (JPMS) added the *module path* (`--module-path`) as a stricter alternative, where modules declare dependencies and exported packages explicitly (`module-info.java`) instead of everything being mutually visible — most existing code still runs on the classpath, and modules are opt-in.

## 3. Linking

Three sub-phases, run against the loaded bytecode before it's usable:

- **Verification**: the bytecode verifier checks the `.class` file is structurally sound and doesn't violate JVM safety rules — no stack over/underflows, correct operand types, no illegal casts, no jumping into the middle of another method, access rules honored. This is what makes running bytecode from an untrasted source safe — a corrupted or hand-crafted malicious `.class` file gets rejected here rather than corrupting the running JVM.
- **Preparation**: memory is allocated for static fields, given default values (`0`/`null`/`false`) — but static initializer *code* hasn't run yet.
- **Resolution**: symbolic references in the constant pool (to other classes/methods/fields) get resolved into direct references — often done lazily, on first actual use, rather than all at once.

## 4. Initialization

Static initializer blocks (`static { ... }`) and static field initializer expressions run now, in textual order, exactly once — triggered the first time the class is "actively used" (an instance created, a static method invoked, a static field accessed or assigned — merely referencing the class elsewhere doesn't count). A superclass is always initialized before its subclass.

## 5. Finding and invoking `main`

The JVM looks specifically for `public static void main(String[] args)` on the named class — that exact signature (public, static, `void`, single `String[]` parameter), or it fails with `NoSuchMethodError`/"main method not found." A new thread — the `main` thread — is created, and it invokes `Demo.main(args)`, with `args` populated from whatever followed the class name on the command line. From here, execution proceeds as ordinary Java code.

## 6. The execution engine — interpreter, then JIT

Bytecode doesn't run natively from the start. The JVM's interpreter executes it one instruction at a time initially — which is why "cold" code (run only a few times) is relatively slow. HotSpot's **tiered compilation** progressively promotes "hot" methods (called or looped often) to native machine code:

- **C1 (client compiler)**: compiles quickly with light profiling and modest optimization — used for early tiers, favoring fast warm-up.
- **C2 (server compiler)**: compiles more slowly but applies heavy optimization (inlining, loop unrolling, escape analysis, dead-code elimination) — reserved for methods proven hot enough across enough invocations.
- If an assumption an optimized method relied on turns out wrong later (e.g. a call site thought monomorphic gets a second implementation loaded), the JVM **deoptimizes** that method back to the interpreter until it can safely recompile.

This interpret-then-JIT pipeline is why Java's startup is comparatively slow relative to natively-compiled languages, while steady-state throughput can end up competitive once the hot paths are fully compiled.

## 7. Program end and shutdown

The JVM keeps running as long as at least one **non-daemon** thread is alive — the `main` thread is non-daemon by default, and any thread you spawn is non-daemon unless explicitly marked otherwise. Once every non-daemon thread finishes (or `System.exit()` is called explicitly), the JVM begins shutdown: it runs any threads registered via `Runtime.getRuntime().addShutdownHook(...)` (concurrently, in no guaranteed order), then terminates — exit code `0` by default, whatever was passed to `System.exit(n)`, or `1` if an exception propagated uncaught out of `main`.

## Real-life analogy

Think of a theater staging a play tonight. `javac` already translated the playwright's script into a universal stage-direction notation shared by every theater (bytecode) — done once, long before tonight's performance. `java Demo` is this specific company actually staging it: someone fetches the notation booklet from the archive (class loading — head librarian → department archivist → this production's own assistant, each checking the shelf above them first, and only fetching booklets actually needed for tonight's scenes, not the whole archive); a safety inspector checks the directions don't contain anything physically impossible (verification); stagehands set every prop to its default position before rehearsal (preparation); references like "the sword from Act 2" get resolved to the actual physical prop only the first time it's genuinely needed (resolution); and the cast runs through entrance blocking exactly once before the show truly opens (static initialization). The performance itself starts precisely at the scene marked "enter protagonist" (`public static void main`) — if no such scene exists in the script, the show can't start at all. During the run, actors read unfamiliar lines a little stiffly off the page at first (the interpreter), but a line delivered every single night (a hot loop) eventually gets memorized and delivered fluidly without the page (JIT-compiled) — and if a late rewrite changes something already memorized, the actor briefly goes back to reading it until re-memorizing (deoptimization). The show only truly ends once every essential cast member has left the stage (non-daemon threads finished) — background crew (daemon threads) never hold up the closing.

## Common gotchas

- **`java Demo` needs `Demo.class`, not `Demo.java`** — except the newer single-source-file launch (`java Demo.java`, JEP 330), meant for quick scripts, not production.
- **Classes load lazily, not all upfront** — an unused, missing dependency can go completely unnoticed until the specific code path that touches it actually runs.
- **Loaded ≠ initialized** — these are distinct, JLS-defined phases; a class can be loaded and verified without its static initializers having run yet, which only happens on first active use.
- **`main`'s signature must match exactly** — `public static void main(String[] args)`. A missing `static`, wrong parameter type, or different casing all mean the JVM reports it can't find a main method, even though a method literally named `main` exists in the class.
