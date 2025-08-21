---
weight: 999
title: "Jetpack Compose: Practical Notes & Pitfalls to Watch For"
description: "If you’re moving UI to Jetpack Compose, you’ll quickly feel the speed and expressiveness. You’ll also discover a few sharp edges that can cost you frames, memory, or correctness. This post is a concise field guide to the things I double-check in every Compose screen: how recomposition actually works, what to store where, how to keep list items stable, and how to avoid leaks."
icon: "article"
date: "2025-08-21T17:41:49+07:00"
lastmod: "2025-08-21T17:41:49+07:00"
draft: false
toc: true
---

# Author : ChungHA (RxMobileTeam)

# Jetpack Compose: Practical Notes & Pitfalls to Watch For

### 1. What is recomposition? When does it happen? How do you detect and optimize unnecessary recomposition?
* **Recomposition** is the process where Jetpack Compose redraws (rebuilds) part or all of the UI tree when the data (state) a composable depends on changes.
* **It happens when:**
    * A state source used by the composable (e.g., `mutableStateOf`, `LiveData`, `StateFlow`…) changes its value.
    * The parameters passed into the composable change.
* **Detecting it:**
    * Use logs or debug tools: `Log.d("Recomp", "Composable X recomposed!")`
    * Use **Layout Inspector** (Android Studio) to track the recomposition count.
* **Optimizing it:**
    * Split composables into smaller pieces and pass only the props they actually need.
    * Use `remember`, immutable objects/classes, and annotate with `@Stable` when appropriate.
    * Avoid passing lambdas/objects that are created on the fly inside the composable body.
    * Avoid creating new lists/collections on every recomposition.

---

### 2. How to use `remember`, `rememberSaveable`, and `derivedStateOf` for different use cases?
* **`remember`:**
    * Stores data / memoized results short-term (within the lifecycle of the current composable tree).
    * Examples: caching an object, a callback, UI state that only needs to be remembered within a small session.
* **`rememberSaveable`:**
    * Persists data across process death and configuration changes.
    * Examples: `TextField` value, form state, scroll position.
* **`derivedStateOf`:**
    * Computes data derived from multiple states and only updates when its input states change.
    * Examples: filtered list, aggregates/totals, a view mode computed from several smaller variables.

---

### 3. When should you use `key` in `LazyColumn`/`LazyList`? What are the risks if you don’t?
* **Use `key` when:**
    * Displaying a dynamic list (items can be added, removed, or moved).
    * Items can change position or content but retain identity.
* **Risks without a key:**
    * Compose can’t tell which item is old/new → may reuse views incorrectly, cause UI flicker, or lose transient state (input, scroll position…).
    * Performance drops due to excessive redraws.
* **Usage:**
```kotlin
items(userList, key = { it.id }) { user ->
    // ...
}
```

---

### 4. How does Compose handle the Slot API and CompositionLocal? Real-world cases for custom CompositionLocals.
* **Slot API:**
    * Compose lets you pass UI blocks into a parent component via lambdas (slots). E.g., custom header, custom button content.
    * Example:
```kotlin
@Composable
fun MyCard(content: @Composable () -> Unit) {
    // ...
}
```
* **CompositionLocal:**
    * Allows passing context or data deep down the tree without threading it through every prop.
    * Use cases: theme, locale, spacing, user/session, permission state…
    * Example:
```kotlin
val LocalSpacing = compositionLocalOf { 8.dp }

CompositionLocalProvider(LocalSpacing provides 16.dp) {
    // children can read LocalSpacing.current
}
```

---

### 5. How to build a custom layout composable? How to optimize heavy layouts in Compose?
* **Custom layout:**
    * Use the `Layout` function, or a `Modifier.layout` to build custom layout logic.
    * Example:
```kotlin
@Composable
fun MyCustomLayout(content: @Composable () -> Unit) {
    Layout(content = content) { measurables, constraints ->
        // Measure children and place them based on custom rules
        // return layout(width, height) { placeables.forEach { it.place(x, y) } }
    }
}
```
* **Optimizing heavy layouts:**
    * Avoid deep, nested layout hierarchies.
    * Prefer standard composables (`Row`, `Column`, `Box`) or well-designed custom layouts.
    * Use `Modifier.layoutId` with `LazyLayout` where applicable.
    * Reuse layout logic; don’t explode into too many tiny composables if it doesn’t help.

---

### 6. Distinguish `@Composable`, `@Stable`, `@Immutable`, `@ReadOnlyComposable`. Impact on performance?
* **`@Composable`**: Marks a function that can participate in the compose tree and is controlled by the Compose runtime.
* **`@Stable`**: Guarantees an object’s observable properties don’t change unexpectedly; helps Compose decide when it can skip recomposition.
* **`@Immutable`**: All properties are `val` and immutable; Compose can safely skip recomposition when the reference is unchanged.
* **`@ReadOnlyComposable`**: For read-only, side-effect-free functions; allows calls from any thread and runtime optimizations.
* **Performance impact:**
    * Correctly applying `@Stable` / `@Immutable` enables Compose to **skip** recompositions and improve performance.
    * `@Composable` is required so Compose can understand and manage the UI function.

---

### 7. Passing mutable objects (lists, classes) into a composable—what to watch for? Optimal solutions?
* **Caveats:**
    * If a mutable object changes without creating a new instance, Compose may not detect it to rebuild the UI.
    * Creating a brand-new object on every recomposition can hurt performance.
* **Solutions:**
    * Prefer immutable objects / `data class` and immutable lists.
    * If you must pass a mutable object, ensure you create a **new** instance when changes occur (e.g., `copy`).
    * Use `@Stable` or `@Immutable` to communicate object characteristics to Compose.

---

### 8. Recomposition, skipping, and invalidation in Compose. When does Compose auto-skip?
* **Invalidation:** When a state or prop a composable depends on changes, Compose marks that region as **invalid**.
* **Recomposition:** Compose calls the invalid composables again to update the UI tree.
* **Skipping:** If Compose determines the inputs haven’t changed (via `equals`/`hashCode`/`@Stable`/`@Immutable`), it can skip recomposition for that composable.
* **Auto-skip occurs when:**
    * Parameters and state values are unchanged (new value equals old value; immutable objects).

---

### 9. When can Compose cause memory leaks? How to detect and prevent them?
* **Leaks can occur when:**
    * Holding references to `Context`/`Activity`/`View`/`Lifecycle` outside the composable’s proper scope.
    * Registering listeners/callbacks without proper teardown (e.g., not removing listeners in `DisposableEffect`).
* **Detection:**
    * Use Profiler, **LeakCanary**, or add debug logs in `DisposableEffect`/`LaunchedEffect`.
* **Prevention:**
    * Keep references alive only within suitable scopes (don’t hold onto `Context` long-term).
    * Use `DisposableEffect` to clean up resources.
    * Prefer `remember` and avoid passing `Context` into globals or long-lived lambdas.

---

### 10. Performance comparison: Compose vs classic Views for very large lists (10k, 100k items). How to measure and optimize?
* **Comparison:**
    * Compose with `LazyColumn` uses lazy loading similar to `RecyclerView`. If you don’t use keys correctly or structure code poorly, it’s easy to trigger excessive recomposition.
    * Classic Views (`RecyclerView`) are highly optimized for large lists via the ViewHolder pattern.
* **Measuring:**
    * Use **Layout Inspector**, **Profiler** in Android Studio, recomposition logs/counters, and track GC.
    * Measure FPS and memory usage while scrolling large lists.
* **Optimizing:**
    * Always provide a stable `key` to `LazyColumn`.
    * Avoid rebuilding item composables; factor items into smaller composables where it helps.
    * Minimize heavy logic/computation inside item composables.
    * Use **Paging** for very large datasets.
