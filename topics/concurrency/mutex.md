---
title: "Mutex: Sharing Versus Transferring State"
---

Swift's `Synchronization.Mutex` stores the state it protects. Multiple tasks can share the mutex, even when the value inside isn't `Sendable`. Its synchronous `withLock` closure gives one caller at a time access to the stored value and lets that caller update or replace it.

## What's the difference between reading, transferring, and leaking?

Imagine a mutex that contains a mutable account object with an integer balance:

- **Read a snapshot:** Read the balance under the lock and return the integer. The caller gets a copy, not access to the account.
- **Transfer the account:** Take the original account out and replace it with a fresh one before releasing the lock. If no other references to the original remain, it can be handed off without sharing it.
- **Leak the account:** Return the original account without replacing it. The caller and the mutex still refer to the same mutable object. A caller can now change it without taking the lock.

```swift
import Synchronization

final class Account {
    var balance = 100
}

func compareAccessPatterns() {
    let mutex = Mutex(Account())

    let snapshot = mutex.withLock { $0.balance }
    print(snapshot) // A copy of the balance.

    let transferred = mutex.withLock { account in
        let original = account
        account = Account()
        return original
    }
    transferred.balance += 50 // The mutex now holds a different account.

    // Don't do this: it returns the replacement without removing it.
    // let leaked = mutex.withLock { $0 }
}
```

That's the distinction behind `sending` in `Mutex.withLock`: a non-`Sendable` value can cross an isolation boundary when it's transferred safely. That doesn't make it safe to share the same object.

## What did the Swift concurrency bug allow?

Swift's region-based isolation checker is meant to track whether references to a non-`Sendable` object are still reachable from more than one place. In the compiler behavior demonstrated, directly returning the stored object from `withLock` compiled even though the mutex still held it. Two tasks could then obtain the same reference and access its mutable state outside the lock. The resulting data race could corrupt data or crash.

That's a hole in the compiler's checks, not a failure of the mutex to serialize its closures. Whether the code compiles can depend on the Swift toolchain; compilation alone isn't proof that the object stayed protected.

### Minimal compiler repro

Apple Swift 6.4.2 accepts this in Swift 6 mode with complete concurrency checking. This was verified by compiling, not by running the race.

```swift
import Synchronization

final class Account {
    var balance = 0
}

func reproduce() {
    let mutex = Mutex(Account())

    Task.detached {
        let account = mutex.withLock { $0 }
        account.balance += 1
    }

    let account = mutex.withLock { $0 }
    account.balance += 1
}
```

Both `withLock` calls return the same account without removing it from the mutex. Each increment happens **after** the lock is released, so the two increments can race.

**What I want to remember:** Share the mutex, not the mutable object inside it. Return a snapshot unless I actually need to hand off the object.

See [Actors vs. Locks: When to Use Each](actors-vs-locks.md) for help choosing between a mutex and an actor.

[Back to Swift Concurrency](index.md)
