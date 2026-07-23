# Kotlin Contracts

Kotlin contracts let a function tell the compiler, in a formal way, something about its own behavior that the compiler couldn't otherwise infer just by reading the function body from the outside — mainly: "if I return in a certain way, then this condition is guaranteed true" or "I call this lambda parameter exactly once." The compiler then uses that declared guarantee to allow smart casts and definite-assignment checks it would otherwise reject.

## The problem contracts solve

Kotlin's compiler can normally only smart-cast a variable to a non-null type when it can *see* the null check itself, right there in the code (`if (x != null) { ... }`). The moment that check happens *inside a function call* — `require(x != null)` — the compiler, from the outside, just sees "a function was called that returned `Unit`." It has no way to know that the function's contract is "if this returns normally, the condition you passed in was true."

```kotlin
// Without a contract on require(), this would be a compile error:
fun withoutContract(x: String?) {
    require(x != null)
    println(x.length) // hypothetically: compile error — x is still seen as String?
}

// With require()'s actual contract (`returns() implies value`), this compiles:
fun withContract(x: String?) {
    require(x != null)
    println(x.length) // smart-cast to non-null String — the compiler trusts require()'s contract
}
```

Contracts are the mechanism that closes that gap — they let the standard library (and your own code) declare that relationship explicitly, so the compiler can trust it.

## The two kinds of contract

**1. `returns(...) implies (condition)`** — describes what a particular return value (or successful, non-throwing completion) implies about some boolean condition. This is what powers `require()`, `check()`, and `isNullOrEmpty()`:

```kotlin
public inline fun require(value: Boolean) {
    contract {
        returns() implies value
    }
    if (!value) throw IllegalArgumentException("Failed requirement.")
}

public inline fun CharSequence?.isNullOrEmpty(): Boolean {
    contract {
        returns(false) implies (this@isNullOrEmpty != null)
    }
    return this == null || this.length == 0
}
```

`require`'s contract reads: "if this function returns normally (doesn't throw), then `value` was true." `isNullOrEmpty`'s contract reads: "if this returns `false`, then the receiver is not null." That's the entire reason `if (!str.isNullOrEmpty()) { str.length }` smart-casts `str` to non-null afterward — `isNullOrEmpty` is a completely ordinary function otherwise, with no special compiler support beyond its declared contract.

**2. `callsInPlace(lambda, InvocationKind)`** — describes how many times (and in what order) a lambda parameter is guaranteed to be invoked. This is what lets `run`, `let`, `apply`, `also`, and `with` treat a `val` assigned inside their lambda as definitely assigned:

```kotlin
public inline fun <R> run(block: () -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}

fun demo() {
    val x: Int
    run { x = 42 }   // legal only because run's contract guarantees the lambda runs exactly once
    println(x)        // x is definitely assigned
}
```

Without that contract, the compiler can't assume a higher-order function calls its lambda parameter at all (it might call it zero times, or several) — so assigning a `val` inside it would normally be rejected as possibly unassigned or possibly reassigned. `InvocationKind` has four values: `EXACTLY_ONCE`, `AT_MOST_ONCE`, `AT_LEAST_ONCE`, and `UNKNOWN`.

## Writing your own

```kotlin
@OptIn(ExperimentalContracts::class)
fun isValidEmail(value: String?): Boolean {
    contract {
        returns(true) implies (value != null)
    }
    return value != null && value.contains("@")
}

fun process(email: String?) {
    if (isValidEmail(email)) {
        println(email.length) // smart-cast to non-null String, thanks to the contract above
    }
}
```

The `contract { ... }` block must be the first statement in the function body. The feature requires `@OptIn(ExperimentalContracts::class)` and has stayed marked experimental for years — because of the caveat below.

## The important caveat: the compiler trusts you, it doesn't verify you

A contract is a **declared promise, not a proven one**. The compiler does not analyze the function body to confirm the contract is actually true — it just takes your word for it and propagates that assumption to every call site. If the contract is wrong (e.g. you write `returns(true) implies (value != null)` but the function can actually return `true` for a `null` value), the code compiles fine and every caller gets an unsound smart cast — which can crash with a `NullPointerException` or similar at runtime, in code that looked perfectly safe.

**Real-life analogy**: a contract is like a signed inspection certificate — "if this certificate says PASS, the building has no structural cracks." Everyone downstream (insurers, buyers) trusts that certificate without personally re-inspecting the building. That's efficient, right up until the inspector signs off on a building that actually does have cracks — the certificate itself doesn't get re-checked against reality, so the false promise flows downstream to everyone who trusted it.

## Where this is most visible day to day

Every time `require()`, `requireNotNull()`, `check()`, `checkNotNull()`, `isNullOrEmpty()`, or a scope function (`let`, `run`, `apply`, `also`, `with`) makes a smart cast or definite-assignment check "just work," a contract declared inside the standard library is the reason — not a special case hardcoded into the compiler for that specific function.
