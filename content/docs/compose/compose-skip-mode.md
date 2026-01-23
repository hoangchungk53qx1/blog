---
weight: 999
title: "Strong Skipping & Lambda Memoization"
description: ""
icon: "article"
date: "2026-01-23T14:08:13+07:00"
lastmod: "2026-01-23T14:08:13+07:00"
draft: false
toc: true
---

# Strong Skipping & Lambda Memoization in Jetpack Compose  
*A clearer explanation of what actually happens under the hood*

Jetpack Compose continues to evolve toward making performance optimizations more automatic and less error-prone for developers. Two important concepts behind this direction are **Strong Skipping Mode** and **lambda memoization**. While they are often mentioned briefly, their interaction is subtle and worth understanding.

This article explains what Strong Skipping Mode really does, how lambda memoization fits into it, and why this matters for real-world Compose code.

---

## What is Strong Skipping Mode?

In Compose, *skipping* means avoiding recomposition of a composable when its output does not need to change. Traditionally, skipping was only possible if **all parameters were stable and unchanged**. If even one parameter was considered unstable, the composable would recompose.

**Strong Skipping Mode changes this rule.**

When Strong Skipping Mode is enabled:

- Restartable composables can be skipped even if they receive unstable parameters
- Compose relies more on runtime comparisons rather than conservative compile-time assumptions
- This significantly reduces unnecessary recompositions

In short, Strong Skipping Mode makes Compose **more aggressive and smarter** about skipping work.

---

## Why Lambdas Were a Problem Before

Lambdas are extremely common in Compose APIs:

```kotlin
Button(onClick = { doSomething() }) { ... }
```
Even though lambdas are considered stable types, they are often recreated on every recomposition. From Compose’s point of view, this means:
* A new lambda instance ≠ the previous lambda instance
* The parameter has “changed”
* The composable cannot be skipped
As a result, lambdas became one of the most common reasons recomposition happened even when UI logic didn’t actually change.

Lambda Memoization with Strong Skipping

With Strong Skipping Mode enabled, the Compose compiler introduces automatic lambda memoization.

Conceptually, this:

```kotlin
val onClick = {
    use(a)
    use(b)
}
```

is transformed into something equivalent to:

```kotlin
val onClick = remember(a, b) {
    {
        use(a)
        use(b)
    }
}

```

What does this mean?
* Lambdas are remembered across recompositions
* A lambda is recreated only when its captured values change
* If captured values are unchanged, the same lambda instance is reused
* This allows Compose to skip recomposition more often
The keys used for memoization depend on the captured variables:
* Stable values → compared using equals
* Unstable values → compared using referential equality (===)
This behavior is automatic — developers no longer need to manually wrap most lambdas in remember.


Why This Is a Big Win
This change brings several benefits:
* Less boilerplate: No need to manually remember callbacks everywhere
* Better default performance: Fewer accidental recompositions
* Safer APIs: Callback-heavy composables (buttons, lists, text fields) become cheaper by default
* Better mental model: “If nothing changed, Compose really won’t redo the work”
In practice, this makes Compose feel closer to how developers intuitively expect UI frameworks to behave.


Opting Out When Needed
Automatic memoization is powerful, but it’s not always desired.

`@DontMemoize`
If you explicitly want a lambda to be recreated on every recomposition (for example, for logging or debugging purposes), you can annotate it with:

`@DontMemoize`
`@NonSkippableComposable`
If a composable must always recompose (for correctness or side-effects), you can prevent skipping entirely using:

`@NonSkippableComposable`
These escape hatches ensure the system stays flexible and predictable.

Key Takeaways

* Strong Skipping Mode allows Compose to skip recomposition even with unstable parameters
* Lambdas are now automatically memoized based on their captured values
* This greatly reduces unnecessary recompositions without manual optimization
* Developers still retain full control through opt-out annotations

Final Thoughts
Strong Skipping and lambda memoization represent an important shift in Jetpack Compose’s design philosophy:performance by default, without sacrificing correctness or developer ergonomics.
For most developers, this means writing simpler code and getting better performance “for free” — while still having the tools to fine-tune behavior when needed.
