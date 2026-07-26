---
title: "What Changed at WWDC27"
description: "@State in SwiftUI is now a Macro"
icon: "article"
date: "2026-06-13T09:00:00+07:00"
lastmod: "2026-06-13T09:00:00+07:00"
draft: false
toc: true
weight: 999
---

# Author : ChungHA 

# What changed at WWDC27: `@State` is now a Macro

Every WWDC season, Apple brings both small and big changes to SwiftUI. Some land with a lot of fanfare on the keynote stage, but others slip in quietly under the hood of the framework — the kind you'd never notice unless you were paying close attention.

One of the "quiet but valuable" changes this time around is: **`@State` is no longer a Property Wrapper — it has become a Macro.**

At first glance this sounds like nothing more than a change in the internal implementation; you don't have to touch a single line of your code. But behind it lies an improvement to performance and initialization behavior that a lot of us have banged our heads against over the years. Let's dig into it together in this post.

## What was `@State` before?

Before, `@State` was a **property wrapper**. If you've ever written SwiftUI, you're definitely familiar with it:

```swift
struct PlayButton: View {
    @State private var isPlaying = false

    var body: some View {
        Button(isPlaying ? "Pause" : "Play") {
            isPlaying.toggle()
        }
    }
}
```

The important thing to remember about SwiftUI is: **a View is a value, and it can be created and re-created many, many times.** Every time the parent view re-renders, your `PlayButton` struct might get initialized from scratch.

So why doesn't `isPlaying` get reset to `false` every single time? Because the value of `@State` doesn't live inside the View struct — it's **managed separately by the framework** and "reconnected" back into the view each time the view is rebuilt.

That's the mechanism that lets your state "survive" through all those view re-creations.

## So what actually changed?

Starting with the new version, `@State` is declared as a **macro** instead of a property wrapper:

```swift
@attached(accessor, names: named(init), named(get), named(set))
@attached(peer, names: prefixed(`_`), prefixed(__), prefixed(`$`))
macro State()
```

Don't panic if that declaration above looks a little hard to swallow. The thing you need to remember is: **from the developer's side, you don't have to change anything at all.** The syntax is exactly the same as before:

```swift
@State private var isPlaying = false
```

The mental model for the prefixes stays the same too:

```swift
@State private var count = 0

// mental model:
count   // wrapped value  - the real value
$count  // binding        - used for two-way passing
_count  // backing storage - the storage behind the scenes
```

The macro automatically generates the accessors (`init`, `get`, `set`) and the peer properties (`_count`, `$count`...) right at compile time. In other words, what the property wrapper used to do "hidden under the hood" is now done more explicitly by the macro at compile time.

> **A quick tie-in to Kotlin so the Android folks can picture it more easily:**
>
> If you're comfortable with Kotlin, you can map this pretty closely:
>
> - The **old Property Wrapper** is very much like a **property delegate** in Kotlin — something like `val isPlaying by remember { ... }` or `var x by Delegates.observable(...)`. Both wrap a value and inject `get`/`set` logic at **runtime**.
> - The **new Macro** is more like a **compiler plugin / KSP (Kotlin Symbol Processing)** — code is **generated right at compile time**. You write `@State`, and the compiler "expands" it into real accessors and a backing field, just like how KSP generates code for Room, Moshi, or how `@Composable` is transformed by the Compose compiler plugin.
>
> And the coolest part — the lazy initialization I'm about to talk about — also has a very familiar "cousin" over in Kotlin: the `by lazy { ... }` keyword. The spirit is identical: **initialize exactly once, on first access, instead of running over and over.**

## The biggest improvement: Lazy Initialization 

This is the part that really makes this change worthwhile.

### The problem before

Let's look at an example with an `@Observable` class:

```swift
@Observable
final class ViewModel {
    var index = .zero

    init() {
        print("Init")
    }
}

struct MyView: View {
    @State private var viewModel = ViewModel()

    var body: some View {
        Button("Index: \(viewModel.index)") {
            viewModel.index += 1
        }
    }
}
```

The problem is in the line `@State private var viewModel = ViewModel()`.

Because a View is a value that gets re-created many times, the expression `ViewModel()` — i.e. the default value — **could get executed again every time the parent view rebuilds this struct.** That means `ViewModel`'s `init()` gets called multiple times, printing `"Init"` over and over, even though in the end SwiftUI only keeps exactly one instance.

For a "lightweight" init, no big deal. But if inside your `init()` you:

- Open a subscription,
- Allocate heavy resources,
- Make a network call, register an observer...

then this is a real waste (and sometimes an actual bug).

### The old "workaround" a lot of people used

To avoid init being called multiple times, a common pattern was to use **optional state** and then initialize it inside `.task`:

```swift
@State private var viewModel: ViewModel?

var body: some View {
    MyView(viewModel: viewModel)
        .task {
            viewModel = ViewModel()
        }
}
```

This works, but it makes the code uglier: you have to handle the optional everywhere, and the initialization logic gets separated from where it's declared.

### After switching to a Macro

With the macro-based implementation, **the default value is initialized lazily** — exactly **once** at the moment SwiftUI creates the storage for the state, and **not** every time the view is rebuilt.

That means with the example above, `print("Init")` now runs exactly once. 🎉

And the optional-state-then-initialize-in-`.task` pattern above is now **no longer necessary** for the case of initializing with a default value. You can just write it naturally:

```swift
@State private var viewModel = ViewModel()
```

and that's enough.

## An important note: initializing in a View's `init` is different

Don't confuse the **default value** with **assigning a value inside the View's initializer**. These two behave differently.

```swift
struct MyView: View {
    @State private var viewModel: ViewModel

    init(id: Item.ID) {
        viewModel = ViewModel(id: id)
    }

    var body: some View {
        // ...
    }
}
```

In this case, the View's `init(id:)` **can still be called multiple times** (because the View is still a value that gets re-created). The lazy initialization improvement above **does not apply** to the assignment in the initializer.

So **don't use this pattern to inject a dependency based on a parameter from the parent** — because you don't control how many times it runs. Lazy init is only "magical" with default values, not with assignments inside `init`.

## A refresher on the state patterns so you use them correctly

While we're on the subject of `@State`, let me sum up when to use what so you don't mix them up:

```swift
// This view owns local value state.
@State private var count = 0

// This view owns an Observable reference.
@State private var book = Book()

// Child can replace the parent's value or reference.
@Binding var book: Book?

// Child needs bindings to the properties of an Observable object.
@Bindable var book: Book
```

A quick breakdown:

- **`@State`**: use it when this view is the owner of the state, whether that's a value type or a reference (`@Observable`).
- **`@Binding`**: use it when the child needs to **replace** the very variable the parent is holding (for example, setting `book = nil`).
- **`@Bindable`**: use it when you already have an `@Observable` object and want to create bindings to the **properties inside** it (for example `$book.title` to feed into a `TextField`).

An example illustrating an `@Observable` passed down to a child without needing a binding:

```swift
@Observable
class Library {
    var name = "My library of books"
}

struct ContentView: View {
    @State private var library = Library()

    var body: some View {
        LibraryView(library: library)
    }
}

struct LibraryView: View {
    var library: Library

    var body: some View {
        Text(library.name)
    }
}
```

Because `Library` is an `@Observable` reference, the child just needs to receive the object directly to observe and mutate its properties — no `@Binding` needed:

```swift
struct BookCheckoutView: View {
    var book: Book

    var body: some View {
        Button(book.isAvailable ? "Check out book" : "Return book") {
            book.isAvailable.toggle()
        }
    }
}
```

Only when the child needs to **swap out the reference itself** (for example, wiping the book back to `nil`) do you need `@Binding`:

```swift
struct ContentView: View {
    @State private var book: Book?

    var body: some View {
        DeleteBookView(book: $book)
            .task {
                book = Book()
            }
    }
}

struct DeleteBookView: View {
    @Binding var book: Book?

    var body: some View {
        Button("Delete book") {
            book = nil
        }
    }
}
```

And when you need bindings to a specific property of an observable object, use `@Bindable`:

```swift
struct BookEditorView: View {
    @Bindable var book: Book

    var body: some View {
        TextField("Title", text: $book.title)
    }
}
```

## Wrapping up

Changing `@State` from a property wrapper to a macro is a textbook example of SwiftUI's philosophy: **keep the developer experience the same on the surface, but improve things dramatically underneath.**

Things to remember:

1. **The syntax doesn't change** — you still write `@State private var ...` just like before.
2. **The default value is now initialized lazily** — it runs exactly once when the storage is created, and no longer gets called again on every view re-render.
3. **The optional-state + `.task` pattern** used to dodge multiple inits is now **no longer necessary** for the default-value case.
4. **Be careful with assignments in a View's `init`** — those still run multiple times, so don't use them for dependency injection.
5. Remember to clearly distinguish **`@State` / `@Binding` / `@Bindable`** so you pick the right tool.

A small change on a WWDC slide, but one that removes one of the most tiresome "workarounds" when working with `@Observable`. Sometimes the most valuable improvements are the quiet ones like this.

---
