---
title: "Swift Concurrency: GCD to async/await, Task, Actor (Part 1 — Basics)"
description: "From GCD to async/await, Task, Actor and the essentials around them"
icon: "article"
date: "2026-07-25T09:00:00+07:00"
lastmod: "2026-07-25T09:00:00+07:00"
draft: false
toc: true
weight: 999
---

# Author: ChungHA

# Swift Concurrency

Everyone doing iOS these days has already touched async/await: shorter code, flatter code, sprinkling `@MainActor` here and there, trying out a few actors, and somehow making those `Sendable` errors go away.

Android devs reading this will recognize the `suspend` style — async, but reads like sync.

But knowing the syntax and understanding the system are two different things. Why won't the compiler let me pass this class between tasks? Why does an actor claim to protect state, yet the state still changes in the middle of my own method? Why does one harmless semaphore freeze the whole app? As long as you keep memorizing each case like that, Swift Concurrency stays a pile of confusing rules.

Every app has a **Main Thread** that handles the UI (buttons, screens, animations). You can also spin up other threads to do work in the background.

### Why does this matter?

- If a heavy task (a network request, image processing, a database read) runs on the Main Thread, the whole app freezes — buttons stop responding, scrolling stops.
- So we push heavy work to the background and keep the Main Thread free, doing nothing but updating the UI.

In the old days, iOS devs handled this manually with:

- **GCD (Grand Central Dispatch)** — `DispatchQueue.global()` and `DispatchQueue.main`.

## How did GCD handle concurrency?

Before Swift Concurrency, iOS used **Grand Central Dispatch (GCD)** to run tasks asynchronously. Devs put tasks onto Dispatch Queues, and GCD automatically managed the threads.

```swift
DispatchQueue.global().async {
    // Do on background
}

// After finishing the background work, the dev had to manually
// hop back to the Main Queue to update the UI.
DispatchQueue.global().async {
    let data = fetchDataFromBackground()

    DispatchQueue.main.async {
        self.label.text = data
    }
}
```

### The downsides of GCD

- When you needed several async tasks, the code nested layer upon layer and became very hard to read.
- The dev had to manually switch back and forth between the background queue and the main queue.
- Forget to switch back to the main queue just once, and the UI could crash or freeze.
- As the project grew, the code got longer and harder to follow because of all the nested closures.

```swift
DispatchQueue.global().async {
    api1()

    DispatchQueue.main.async {
        api2()

        DispatchQueue.global().async {
            api3()
        }
    }
}
```

**In a later part I'll dig deeper into how Swift Concurrency differs from GCD at the thread level.**

## The core problem

GCD only knows one thing: *"run this piece of code on some thread."* That's it. It has no idea:

- What data your code is touching.
- Whether that same data is being touched by another thread at the same time.
- Whether two threads are accessing that data at the exact same moment.

### What is a data race?

A **data race** happens when two different background tasks both try to read and write to **the exact same variable** in memory at **the exact same moment — possibly a fraction of a microsecond apart**.

**An analogy I often use with students — the "double bank transaction":**
- Imagine a husband and wife share a bank account with $100.
- At the exact same millisecond, the husband withdraws $50 at an ATM in Hanoi, while the wife spends $70 at a CircleK store.
- If the banking system has no "lock" or "queue," both transactions read the balance as $100. Both get approved. The result: the balance goes negative, and the database gets cooked.

### Why is it dangerous?

A data race leads to wrong numbers, corrupted UI state (e.g. showing the wrong transfer status), and memory corruption — which can happen especially when working with arrays.

### The core idea

- In the old days, we protected data by using `DispatchQueue` manually.
- **The problem:** the compiler had no idea what we were doing. Forget to use `DispatchQueue.main.async`, or forget a lock in just **one** background function, and a data race occurs.
- These bugs are **invisible**. They pass tests on your machine, pass QA, and only go wrong on the customer's phone — right in the middle of an important money transfer.
- **The win with Swift Concurrency:** it moves this safety check from **runtime** (on the user's device) to **compile-time**. If there's a risk of a data race, the app **won't build**.

### Example

```swift
class DangerousAccountBank {
    var balance: Double = 1000.0

    func withdraw(amount: Double) {
        // ❌ PROBLEM: If two background threads run this at the same time,
        // both read balance = 1000, both get approved,
        // and you overdraw the account without any lock!
        if balance >= amount {
            balance -= amount
        }
    }
}

let account = DangerousAccountBank()

// Thread A: withdraw 700 from Hanoi
DispatchQueue.global().async {
    account.withdraw(amount: 700.0)
}

// Thread B: withdraw 500 from CircleK
DispatchQueue.global().async {
    account.withdraw(amount: 500.0)
}

// Result: the app builds with ZERO warnings, yet a silent
// data race occurs and the balance ends up wrong.
```

## Why modern Swift Concurrency?

At **WWDC 2021**, Apple introduced a new system — **async/await, Task, and Actor** — with three goals:

- Write code that reads top-to-bottom, in a straight line, even though it actually runs asynchronously.
- Let the compiler catch race conditions automatically, instead of relying on the dev to be careful.
- Make error handling simpler and more consistent.


### async and await — the foundation

- **`async`**: placed before a function that takes time to finish (e.g. a callApi). It means "this function won't return a result immediately — it will pause and come back with one."
- **`await`**: used when calling such a function. It means "pause here until the result is ready, but don't block anything else in the app while waiting."

```swift
func fetchUserName() async -> String {
    // Imagine a network call happening here
    return "Ali"
}

@MainActor
func showProfile() async {
    let name = await fetchUserName()
    print("The user's name is: \(name)")
}
```

`await fetchUserName()` means "wait for the result, but don't block the thread."

**The most important point:** `await` does **NOT** block the Main Thread. In the old GCD style the dev had to switch threads manually — now the compiler handles that for you.

### Task — kicking off asynchronous work

An `async` function only runs when something calls it from an async context. To kick off async work from a normal place (e.g. a button tap), we use `Task`:

```swift
Button("Load Profile ChungHA") {
    Task {
        let name = await fetchUserName()
        self.userName = name
    }
}
```

`Task { }` creates a new "packet of async work" that can run in the background, and inside it you can freely use `await`.

## Structured Concurrency — organizing the work

Suppose you need two things at once: the user's name and their profile picture. They're independent, so running them concurrently is better than doing one and then the other.

--> just like async/await in Coroutines.

### async let — concurrency

```swift
func loadProfile() async {
    async let name = fetchUserName()
    async let image = fetchProfileImage()

    let finalName = await name
    let finalImage = await image
}
```

Here, `fetchUserName()` and `fetchProfileImage()` both start at the same time (concurrently), and we only `await` them when we actually need the results. This is much faster than doing them sequentially.

### TaskGroup — when the number of tasks isn't fixed

If you need to load a dynamic number of things (e.g. a list of 10, 20, or more images), use a **TaskGroup** — it runs them all concurrently and collects the results as each one finishes.

```swift
func downloadAllImages() async {
    await withTaskGroup(of: String.self) { group in
        for id in 1...5 {
            group.addTask {
                await downloadImage(id: id)
            }
        }

        for await image in group {
            print(image)
        }
    }
}
```

--> Honestly, this syntax looks a bit dreary to me.

## How does an actor protect shared state?

If we have a shared manager class (e.g. `TransactionManager`), we can't let multiple tasks touch it freely.

**Analogy:** an `actor` is like **a single bank teller** sitting behind a glass window.

- You (Thread A) can't just climb over the counter and grab the cash drawer.
- You have to stand in line and ask the teller.
- If someone else (Thread B) also wants cash, they have to wait behind you.

### Example

```swift
class MyBankAccount {
    var balance = 1000

    func withdraw(amount: Int) {
        if balance >= amount {
            balance -= amount
        }
    }
}

let account = MyBankAccount()

DispatchQueue.global().async {
    account.withdraw(amount: 700)
}

DispatchQueue.global().async {
    account.withdraw(amount: 700)
}

// Thread 1 reads balance -> Balance = 1000
// Thread 2 reads balance -> Balance = 1000
// Thread 1: 1000 - 700 = 300
// Thread 2: 1000 - 700 = 300
// In this case the result can come out as -400, completely wrong
// -> this is exactly a race condition
```

### The solution with an actor

```swift
actor MyBankAccount {
    var balance = 1000

    func withdraw(amount: Int) {
        if balance >= amount {
            balance -= amount
        }
    }
}

let account = MyBankAccount()

Task {
    await account.withdraw(amount: 700)
}

Task {
    await account.withdraw(amount: 700)
}

// Balance = 1000, withdraw 700 -> Balance = 300
// Next task: 700 > 300 -> withdrawal fails
// Flow: Task 1 -> Actor -> Task 2 (serialized, safe)
```

An actor guarantees that only one task touches its internal state at a time, so a data race simply can't happen.

## Why do we need `@MainActor` on specific functions?

**The idea:**

- In iOS, every UI update **must** happen on the Main Thread. If you update a SwiftUI `@Published` property or a `@State` variable from a background thread, the app will glitch or crash.
- **Analogy:** `@MainActor` is like the stage manager of a theater. Only the stage manager is allowed to move the props and change the lights (i.e. the UI) on stage.
- If a background actor (a background thread fetching a bank statement) wants to update the screen, it has to hand the data to the stage manager (`@MainActor`) to put it on screen.
- In Swift, the **Main Actor** is a special actor that runs its tasks on the main thread — the very thread that updates your app's UI.
- The Main Actor makes sure any code that changes the UI runs safely on the main thread.

### Why we mark specific functions with `@MainActor`

If your ViewModel fetches API data on a background thread, then the function that actually **assigns** that data to UI properties (e.g. `self.accountBalance = newBalance`) must be marked `@MainActor`. This guarantees the UI update automatically "hops" back to the main thread, safely.

Example for iOS 16+, whereas on iOS 17 you don't need ``ObservableObject``.

```swift
class BankViewModel: ObservableObject {
    @Published var amount: String = "Loading..."

    func fetchAmount() async {
        let result = await callAmountAPI()
        amount = result // No guarantee this runs on the main thread
    }
}
```

```swift
class BankViewModel: ObservableObject {
    @Published var amount: String = "Loading..."

    @MainActor
    func fetchWeather() async {
        let result = await callAmountAPI() // this part runs on a background thread
        // now the line below is guaranteed to run on the main thread
        amount = result
    }
}
```

--> Doesn't this feel like context preservation in Coroutines?


## Sendable — telling the compiler a type is safe to share

When data is "sent" from one thread to another (e.g. inside a `Task`), Swift needs to be sure that type is thread-safe. This is expressed through the `Sendable` protocol. If a type isn't `Sendable` but gets used across multiple threads, the compiler warns or errors — catching potential bugs before they ever happen.

```swift
struct Profile {
    var name: String
    var age: Int

    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

func processProfile() async {
    let profile = Profile(name: "ChungHA", age: 28)

    Task {
        // The compiler will emit a WARNING/ERROR:
        // "Capture of 'profile' with non-sendable type 'Profile'
        //  in a '@Sendable' closure"
        print(profile.name)
    }
}
```

A `struct` is already sendable when used internally. But if it's used outside the module, you should declare `Sendable` explicitly:

```swift
public struct Profile: Sendable {
    var name: String
    var age: Int
}

func processProfile() async {
   let profile = Profile(name: "ChungHA", age: 28)

    Task {
        print(profile.name) // no more warning
    }
}
```

## What is `nonisolated`?

**The idea:**

- Sometimes Xcode keeps warning *"You cannot access this actor property synchronously."* But you know that property is safe, because it's a constant (a `let`) or just some simple helper function.

- `nonisolated` is how you tell Swift: "This specific function or property doesn't touch any mutable data. Just let anyone read it right away, no `await` needed."

Without `nonisolated`, you'd have to `await` as usual, of course.

**I'll cover this more carefully in Part 2 — actors and `nonisolated` aren't as simple as you might think.**

### Example

```swift
actor BankBranch {
    let code: String = "1998"      // Constant
    var vaultCash: Double = 1000000.0    // Mutable state

    // Marked nonisolated because it only reads the constant code
    nonisolated func getCodeDetails() -> String {
        return "Code: \(code)" // no need to await
    }
}
```

## Quick recap

| Concept | Role |
|---|---|
| **async / await** | Write async code that reads like sequential code, without blocking the Main Thread |
| **Task** | Kick off async work from a normal sync context (a button tap...) |
| **async let** | Run a fixed number of independent tasks concurrently |
| **TaskGroup** | Run a dynamic number of tasks concurrently, collecting results as they finish |
| **actor** | Protect shared state; only one task touches it at a time -> prevents data races |
| **@MainActor** | Guarantee UI-updating code runs on the Main Thread |
| **Sendable** | Mark a type as safe to pass back and forth between threads |
| **nonisolated** | Allow reading an actor's immutable data without `await` |

To sum up, the way I read it: the core idea of Swift Concurrency is to move the "be careful with threads" burden off the dev's shoulders (at runtime) and onto the compiler (at compile-time). You write code as straight as your thoughts, and the compiler handles the safety part.

Stay tuned for Part 2, where we'll dig even deeper.

---
