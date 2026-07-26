---
title: "ImmutableList vs List in Jetpack Compose"
description: "Rethinking the best practice now that we have Strong Skipping Mode"
icon: "article"
date: "2026-06-17T10:00:00+07:00"
lastmod: "2026-06-17T10:00:00+07:00"
draft: false
toc: true
weight: 999
---

# Author : ChungHA 

# ImmutableList vs List in Jetpack Compose: Rethinking the "best practice" after Strong Skipping Mode

Anyone who's been doing Compose for a while probably knows this one by heart: "If you want a composable to skip recomposition, don't pass in a `List<T>` — you've got to use `kotlinx`'s `ImmutableList<T>`."

That advice isn't wrong — at least it wasn't back when it first appeared. But ever since **Strong Skipping Mode** became enabled by default, this rule of thumb has changed. So today I want to sit down and go through each case, one by one, to see whether `ImmutableList` is still a "must-have" or not.

In this article I'll break down 3 concrete scenarios, along with the real cost each choice actually pays.

## A quick recap: why did we always reach for ImmutableList back then?

In Compose, a composable can only **skip** (i.e. skip recomposition) when all of its parameters are both **stable** and **unchanged**.

The catch is this: `List<T>`, `Set<T>`, and `Map<T>` in Kotlin are all treated as **unstable** by the Compose compiler. Why? Because the `List<T>` interface makes no promise about immutability — behind a `List` there could very well be a `MutableList` being mutated somewhere behind your back. The compiler has no way of knowing for sure, so to be safe it just assumes it's unstable.

```kotlin
// Before Strong Skipping: MyList can NOT skip
// because the items parameter of type List<T> -> treated as unstable
@Composable
fun MyList(items: List<Item>) {
    Column {
        items.forEach { ItemRow(it) }
    }
}
```

Just one unstable parameter is enough to make the whole composable lose its ability to skip. So people came up with the idea of using a type that commits to immutability from kotlinx

```kotlin
import kotlinx.collections.immutable.ImmutableList
import kotlinx.collections.immutable.toImmutableList

@Composable
fun MyList(items: ImmutableList<Item>) { // stable -> can skip
    Column {
        items.forEach { ItemRow(it) }
    }
}

// call site
MyList(items = viewModelItems.toImmutableList())
```

`ImmutableList` is marked as stable, so the composable gets its ability to skip back. That's the whole story behind that old best practice.

## What did Strong Skipping Mode change?

> If you're not clear on how Strong Skipping works, you should read the [Strong Skipping & Lambda Memoization](../../compose/compose-skip-mode/) article first so it's easier to follow along.

In short: back then, just **one** unstable parameter was enough for a composable to lose its right to skip. **Strong Skipping Mode** flips that rule:

- A restartable composable **can still skip** even when it receives unstable parameters.
- Compose now trusts **runtime comparison** more, instead of guessing conservatively right at compile time.

Specifically, for an unstable parameter (like `List<T>`), Compose compares using **instance equality** — that is, reference comparison (`===`). If the same instance is passed in again, the composable happily skips.

This is exactly what makes the old belief — "unstable collections are always bad" — **no longer true in every case.** And that's really the whole point of today's article.

## Case by case

To be fair, I'll look at both exactly the way Compose works:

- **`List<T>`** → Compose compares with **instance equality** (`===`), dirt cheap, O(1).
- **`ImmutableList<T>`** → since it's treated as stable, Compose compares with **structural equality** (`equals()`), costing O(N). Plus you also pay the extra **convert** cost of `toImmutableList()`, which is O(N) too.

### Case 1: The list never changes (still the same instance)

This is when the list is created once and then kept exactly as-is for its entire lifetime.

```kotlin
// list keeps the same instance across every recomposition
val items = remember { loadStaticItems() } // List<Item>
MyList(items = items)
```

- `List<T>`: instance unchanged → `===` is true → **skip**. Cost O(1).
- `ImmutableList<T>`: it can skip too, but you have to eat one extra `toImmutableList()` convert costing O(N).

**Verdict:** Both skip. `ImmutableList` gains nothing extra, and on top of that it costs one more convert (luckily it's only once, so it's not a big deal). **Call it a tie, but leaning toward `List`.**

### Case 2: The instance changes when the content actually changes

This is the "healthy, common" scenario, and the most common one: when the content changes, a new list is created; when the content stays the same, the old instance is kept.

```kotlin
// Only when the data actually changes does a new list get born
val items: List<Item> by viewModel.items.collectAsState()
MyList(items = items)
```

- `List<T>`: instance changed → `===` is false → recompose (exactly what we want, since the content really is different). Cost O(1).
- `ImmutableList<T>`: has to convert O(N) + compare `equals()` O(N), and in the end **still has to recompose** because the content really is different.

**Verdict:** Recomposing here is mandatory. `ImmutableList` only adds the extra convert plus structural comparison at O(N) — **all of it wasted effort.** `List<T>` still wins.

### Case 3: The content does NOT change but a new instance keeps getting created

This is the **only** place where `ImmutableList` actually gets to shine.

```kotlin
// Every recompose maps out a NEW list even though the content is identical
@Composable
fun Screen(state: UiState) {
    // .map { } spawns a new List instance every time -> === always false
    MyList(items = state.rawItems.map { it.toUi() })
}
```

- `List<T>`: a fresh instance every time → `===` always false → unnecessary recompose even though the content didn't change.
- `ImmutableList<T>`: thanks to `equals()` it recognizes the content is identical → **skips** the upper part of the tree. This is a genuine benefit.

But — and this "but" is really important:

- You still have to eat the **convert** O(N) + `equals()` comparison O(N) every time.
- If the child composable is **lazy** (like `LazyColumn`), the amount of work is already limited to the currently visible region, so the "skip the top" benefit is often **not worth** the O(N) cost you pay.

```kotlin
// With LazyColumn, items only compose according to the visible region
// so ImmutableList's upstream-skip benefit shrinks a lot
LazyColumn {
    items(uiItems) { ItemRow(it) }
}
```

**A worthwhile tip:** instead of forcing yourself to use `ImmutableList`, **pull Case 3 back to Case 2** right at the source — don't spawn a new instance when the content hasn't changed. For example, with `StateFlow` + `distinctUntilChanged()`:

```kotlin
val items: StateFlow<List<Item>> =
    repository.itemsFlow
        .map { it.toUiItems() }
        .distinctUntilChanged() // filter out instances that are duplicate in content
        .stateIn(scope, SharingStarted.WhileSubscribed(5_000), emptyList())
```

Or even simpler, just `remember` with the right key so you don't re-map for no reason:

```kotlin
// but rarely
val uiItems = remember(state.rawItems) { state.rawItems.map { it.toUi() } }
```

Once the instance is stable, you're back to Case 1/2, and at that point `List<T>` is usually more than enough.

## Summary table

| Case | `List<T>` | `ImmutableList<T>` | Who wins |
|---|---|---|---|
| **Case 1** – same instance, no change | skip, O(1) | skip, + one-time convert O(N) | `List` (tie, leaning `List`) |
| **Case 2** – instance changes when content changes | recompose, O(1) | recompose, + convert & `equals()` O(N) | **`List`** |
| **Case 3** – content unchanged, new instance every time | wasteful recompose | skip thanks to `equals()`, but costs O(N) | Depends — `Immutable` only wins when it's not lazy |

## Wrapping up

Ever since Strong Skipping Mode became the default, going out of your way to convert `List<T>` to `ImmutableList<T>` just to "get skipping back" **is no longer necessary in every case.**

The key points to remember:

1. **Strong Skipping** lets a composable skip even with unstable parameters like `List<T>`, based on instance equality (`===`).
2. **Case 2** (instance changes when content changes) is the most common one, and here `List<T>` is always the better choice — `ImmutableList` just wastes effort.
3. `ImmutableList` is only genuinely beneficial in **Case 3**, and even Case 3 should be handled at the source (`distinctUntilChanged`, `remember` with the right key) to pull it back to Case 2.
4. With `LazyColumn`, `ImmutableList`'s benefit shrinks even further.
5. If you're **already** using `ImmutableList`, there's no need to rush to drop it — it doesn't do any harm, it's just no longer mandatory.

The final philosophy sounds very down-to-earth: **don't be tunnel-visioned.** Don't just assume `ImmutableList` is mandatory for Compose performance. **Profile** properly, find the real bottleneck first, and only then optimize in the right spot, folks.

---
