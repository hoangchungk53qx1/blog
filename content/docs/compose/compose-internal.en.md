---
title: "Compose Runtime"
description: "Compose Compiler, Compose Runtime, and the Slot Table in Compose"
icon: "article"
date: "2025-08-14T00:27:57+01:00"
lastmod: "2025-05-14T00:27:57+01:00"
draft: false
toc: true
weight: 999
---

# Today we're going to dig into the Compose Compiler, the Compose Runtime, and the Slot Table in Compose

# Author : ChungHA (RxMobileTeam)

## Compose Compiler, Runtime, and the Slot Table

## I. Introduction

We're going to explore the Compose Runtime, one of the key pieces behind building user interfaces with Jetpack Compose. The Compose Runtime manages how Composables are rendered and updated on
the UI. It uses a structure called the "Slot Table" to keep track of Composables and their state.

Let's quickly recap a bit about Compose first.

1. What is a Composable?
A Composable is a function in Jetpack Compose that lets you define your UI in a declarative way.
Each Composable can contain other components and can be combined to build up complex interfaces.

For example:
```
Text, Column, Button, LazyColumn
```

2. What is the Compose Compiler?
- The Compose Compiler is a part of Jetpack Compose that helps turn Composable functions into machine code.
- It handles compiling Composable functions and generates the corresponding code to render the UI.

3. What is the Compose Runtime?
- The Compose Runtime is a part of Jetpack Compose that's responsible for managing how Composables are displayed and updated on the UI.
- It uses a structure called the "Slot Table" to keep track of Composables and their state, making sure the UI is always updated efficiently.
- To put it more simply, it's like a director, making sure the actors (Composables) are in the right place and playing their roles correctly on the stage (the UI).
- It's constantly watching for changes in a Composable's state and updating the UI whenever it's needed.

4. What is the Slot Table?
- The Slot Table is a data structure inside the Compose Runtime, used to manage Composables and their state.
- It lets the Compose Runtime track Composables and update them efficiently whenever the state changes.
- The Slot Table holds "slots" (positions) for each Composable, and each slot can hold a Composable or a state value.
- The Slot Table helps the Compose Runtime figure out which Composables need to be updated when the state changes, which cuts down on unnecessary redraws of the UI.

## II. Diving deeper into the Compose Runtime and the Slot Table

Compose Runtime: the conductor of Recomposition
Let's move from the clever craftsmanship of the Compose Compiler over to the powerful engine that brings everything to life.
If the compiler is the hidden member of the stage crew, the one quietly rearranging the script behind the scenes, then the runtime is the stage director, making sure every note is played at the right moment and in the right order.
The runtime is driven by state. It doesn't just update the UI when a change happens, it does so in an extremely efficient and smart way.
The runtime pulls off this feat by using structures like the SlotTable. The beauty of the Compose Runtime lies in its ability to orchestrate every action in response to state changes while still keeping performance high.

But how does the runtime achieve this? What goes on underneath that makes it so smooth, so efficient, and most importantly, so responsive?

- Imagine the Compose Runtime constantly observing the Composables, like an attentive supervisor watching every actor on the stage. When the state changes, the runtime figures out which Composables need updating and does it efficiently.
  So what's the payoff here? The runtime doesn't just keep the UI up to date, it also minimizes unnecessary redraws, which improves the app's performance.

The example below illustrates how the Compose Runtime works with the Slot Table:

```kotlin
var saySomething by remember { mutableStateOf("") }
```

- In this example, `saySomething` is a state variable managed by the Compose Runtime. When the value of `saySomething` changes, the Compose Runtime figures out which Composables depend on this variable and updates it.
  What is `remember`? `remember` is a function in the Compose Runtime that lets you store state inside a Composable. When the Composable is called again, the value of `saySomething` is preserved, which avoids recreating the value every time the Composable is redrawn.
  That's the simple way we tend to think about `remember`, but in reality it's a part of the Compose Runtime.
- Under the hood, it doesn't just hold onto a value; it also ties that value to a slot, a memory location that Compose uses to track state across recompositions.
- When the state doesn't change, the runtime doesn't need to recreate anything; it simply reuses the memory location that's already holding that value.
- This helps reduce memory usage and boost performance, since there's no need to recreate objects unnecessarily.
  Let's take a look at the internals of `remember`:


```
@Composable
inline fun <T> remember(
    crossinline calculation: @DisallowComposableCalls () -> T
): T = currentComposer.cache(false, calculation)
```

```
inline fun <T> cache(invalid: Boolean, block: () -> T): T {
    var result = nextSlotForCache()
    if (result === Composer.Empty || invalid) {
        val value = block()
        updateCachedValue(value)
        result = value
    }

    @Suppress("UNCHECKED_CAST")
    return result as T
}
```

As they say, `Talk is cheap, show me the code!`
- In the code above, `remember` uses `currentComposer.cache` to store the value of `saySomething`.
- If the value was stored previously and hasn't changed, the Compose Runtime reuses that value without recreating it.
- If the value changes, it calls `calculation` to compute the new value and updates the corresponding slot in the Slot Table.
  Simply put: `cache` makes sure the value is stored in a specific memory location. If the state doesn't change, that location stays the same. If it does change, that location gets updated.

### Slot Table: the data structure behind the Compose Runtime

The Slot Table is an important data structure in the Compose Runtime that helps manage Composables and their state.
It works like a table, where each row represents a Composable or a state value.
Each Composable is mapped to a slot in the Slot Table, and each slot can hold a Composable or a state value.


{{< figure src="../slot_table.png" alt="Open Ad" width="500px">}}

This isn't your ordinary data structure, it's the secret behind Compose.

- It records memory in a highly optimized Tree-shaped structure, keeping track of where each part of the UI has been and will be during recomposition.
- This Tree-shaped structure is really useful because it mirrors the UI hierarchy, allowing specific parts of the UI tree to be updated efficiently.

So how does this actually work?

- When a composable is executed for the first time, the Compose Runtime walks through it and saves the slots in the SlotTable.
- These slots capture everything from parameters to remembered state values, and even SideEffects.
- Then, when it's time for recomposition, it revisits these slots — not to rebuild them, but to reuse, skip, or update them.

So what is a slot in the Slot Table?
- Each slot in the Slot Table represents a Composable or a state value.
- A slot is an abstract representation of a position in the composition tree.
- Each slot stores data that could be a remembered value, a group of slots, or Nothing :pray

In short: a sequence of slots is grouped together into one big group, and when Compose performs recomposition, it walks through these slots and decides whether they need updating, then applies exactly the updates that are needed.

--> In the actual implementation, Compose uses a complex array-based structure to optimize performance.

Link: Go give it a read :v
```
https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/SlotTable.kt
```

The concrete implementation is at the link, I'm just summarizing here :pray

```agsl
internal class SlotTable : CompositionData, Iterable<CompositionGroup> {
     /**
     * Tracks the number of active readers. A SlotTable can have multiple readers but only one
     * writer.
     */
    private var readers = 0

    /**
     * Tracks whether there is an active writer.
     */
    internal var writer = false
        private set
    /**
     * An internal version that is incremented whenever a writer is created. This is used to
     * detect when an iterator created by [CompositionData] is invalid.
     */
    internal var version = 0
  
     /**
     * An array to store group information that is stored as groups of [Group_Fields_Size]
     * elements of the array. The [groups] array can be thought of as an array of an inline
     * struct.
     */
    var groups = IntArray(0)
        private set
        
    /**
     * The number of groups contained in [groups].
     */
    var groupsSize = 0
        private set
        
    /**
     * An array that stores the slots for a group. The slot elements for a group start at the
     * offset returned by [dataAnchor] of [groups] and continue to the next group's slots or to
     * [slotsSize] for the last group. When in a writer the [dataAnchor] is an anchor instead of
     * an index as [slots] might contain a gap.
     */
    var slots = Array<Any?>(0) { null }
        private set
        
    /**
     * The number of slots used in [slots].
     */
    var slotsSize = 0
        private set
        
    /**
     * A list of currently active anchors.
     */
    internal var anchors: ArrayList<Anchor> = arrayListOf()
}
```
- We can see that it doesn't use a Data Class, it uses an Array to store the slots.
  This helps increase performance and reduce the overhead of using complex objects.
  Since it's a flat array, operations like insert and read are fast, O(1) or close to O(1).
- Each group in the groups array helps Compose determine the position and content of a composable.
- When recomposition happens, it checks whether these groups have been moved, whether a remembered value has changed, whether it's still valid, or whether it's Empty.
- If it determines that a group has been moved, it updates the corresponding slots in the Slot Table.
- If a group can be skipped (for example, when nothing has changed), it doesn't need to update the slots in that group, which saves time and resources.
- Otherwise, it updates the corresponding slots in the Slot Table and moves on:

- `readers` and `writer` are used to manage concurrent access to the Slot Table.
- Only one writer can be active at a time, while there can be many readers.
- This ensures that updating the Slot Table is safe in a multi-threaded environment.


### Slot Reuse: reusing slots in the Slot Table
- When you call remember { mutableStateOf(...) }, Compose doesn't just create a new state holder out of nowhere.
- It checks the current slot position in the SlotTable. If there's a value already cached and the slot group hasn't changed, it reuses that value; otherwise, it writes a new one.

```agsl
/**
     * Read the slot table in [block]. Any number of readers can be created but a slot table cannot
     * be read while it is being written to.
     *
     * @see SlotReader
     */
    inline fun <T> read(block: (reader: SlotReader) -> T): T =
        openReader().let { reader ->
            try {
                block(reader)
            } finally {
                reader.close()
            }
        }

    /**
     * Write to the slot table in [block]. Only one writer can be created for a slot table at a time
     * and all readers must be closed an do readers can be created while the slot table is being
     * written to.
     *
     * @see SlotWriter
     */
    inline fun <T> write(block: (writer: SlotWriter) -> T): T =
        openWriter().let { writer ->
            var normalClose = false
            try {
                block(writer).also { normalClose = true }
            } finally {
                writer.close(normalClose)
            }
        }

```

The code above reads from and writes to the Slot Table, using the `openReader()` and `openWriter()` functions.
- `openReader()` creates a `SlotReader`, which lets you read values from the Slot Table.
- `openWriter()` creates a `SlotWriter`, which lets you write values into the Slot Table.

- When you call `remember`, Compose uses the `SlotWriter` to write the value into the Slot Table.
- If the value already exists in the Slot Table, it reuses that value instead of creating a new one.
- But it's not just caching a value, it's a highly precise, structured form of reuse.
- During recomposition, Compose moves the `reader` across the Slot Table, and it also creates a `writer` so it can selectively replace the slots that need it and skip the ones that don't.

To keep these groups consistent, it uses a class called Anchor.

```agsl
/**
 * An [Anchor] tracks a groups as its index changes due to other groups being inserted and removed
 * before it. If the group the [Anchor] is tracking is removed, directly or indirectly, [valid] will
 * return false. The current index of the group can be determined by passing either the [SlotTable]
 * or [SlotWriter] to [toIndexFor]. If a [SlotWriter] is active, it must be used instead of the
 * [SlotTable] as the anchor index could have shifted due to operations performed on the writer.
 */
internal class Anchor(loc: Int) {
    internal var location: Int = loc
    val valid
        get() = location != Int.MIN_VALUE

    fun toIndexFor(slots: SlotTable) = slots.anchorIndex(this)

    fun toIndexFor(writer: SlotWriter) = writer.anchorIndex(this)

    override fun toString(): String {
        return "${super.toString()}{ location = $location }"
    }
}
```

These anchors, tied to group IDs, let Compose maintain references to specific groups even when the table changes during recomposition,
ensuring stability and efficiency while the UI is being updated.
This is how it tracks the exact position of a composable inside the slot table.

In summary:
The SlotTable is what enables Compose to:
- Reuse the exact memory locations for values like remember across recompositions.
- Skip entire parts of the composition tree when the inputs haven't changed.
- Efficiently restore the UI after configuration changes, or skip recomposition altogether.
- Without the SlotTable, Compose would have to re-run every composable, re-initialize every piece of state, and re-launch every side effect.

And as UI complexity grows, Compose needs ways to stay efficient. This is where Group Introspection and Anchor Stability come into play.
- Group Introspection lets Compose inspect the structure of a composition group without having to re-run its code.
- Instead of executing the group blindly, Compose analyzes the group's keys and boundaries to better understand how the Composables are organized and connected.

By analyzing the group's keys and boundaries, it can:
- Determine whether a group can be skipped entirely.
- Optimize memory allocation for stable structures.
- Anchor Stability is the mechanism that ensures consistent identity of UI elements across recompositions. When working with lists or dynamic content, stability is a critical factor for performance.

For example:
```agsl
LazyColumn {
    items(players) { player ->
        PlayerView(player)
    }
}
```
- Without Anchor Stability, every time the `users` list changes, Compose would have to recreate the entire list, leading to poor performance.
- Recomposition could unnecessarily recreate all the items even when only one item changed.

So how does Compose make sure that only the elements that actually changed get updated?

- Compose uses a mechanism called a "key" to identify elements in a list. Each element has a unique key,
- And when the list changes, Compose compares the keys to figure out which element changed.

```agsl

LazyColumn {
    items(players, key = { player -> player.id }) { player ->
        PlayerView(user)
    }
}
```

==> That wraps up the Slot Table, Slot Reuse, and the Compose Runtime, so the question that comes up is:
How does it know when the UI needs updating, when it doesn't, and how does it avoid unnecessary redraws?
That's when the Recomposer steps in.

### Recomposer
The SlotTable — the storage of state.
- The Recomposer is a part of the Compose Runtime that's responsible for figuring out when the UI needs to be updated.
- The Recomposer is responsible for driving changes through the composition based on state changes or external triggers (like LaunchedEffect, mutableStateOf, or snapshotFlow).
- But the magic lies in how it composes these changes in sync with the SlotTable, simply by using a Tree traversal algorithm.


Here's the sequence of events that happens when a state changes in Compose:

1. A state holder (e.g., MutableState) updates.
2. It notifies the Snapshot system (which manages concurrent read/write access to state).
3. The Snapshot marks certain composition scopes as invalid.
4. The Recomposer receives the invalid scopes and starts a recomposition pass.
5. It traverses the SlotTable with a SlotReader, comparing group keys.
6. It chooses to execute only the functions whose inputs have changed.
7. The results are written back into the SlotTable through the Composer, just like I explained above.

You can think of the Recomposer as a loop: it lives inside a coroutine loop, waiting for changes to loop over.

## III. Conclusion
That was my high-level overview of the Compose Runtime and the Slot Table in Jetpack Compose.
Thanks everyone for reading this far. If I got anything wrong, please let me know so I can make it better.
