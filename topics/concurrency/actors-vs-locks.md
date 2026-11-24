---
title: "Actors vs. Locks: When to Use Each"
---

Start with the API: **Can callers suspend, or does the work need to finish before a synchronous call returns?**

## Choose an actor for shared state behind an async boundary

A [bank actor](actors.md) can own both accounts and debit one and credit the other in a single synchronous `transfer` method. Callers outside the actor use `await` and can suspend while waiting for access instead of blocking a thread. Swift checks the isolation boundary, and the actor's synchronous methods can call one another without taking another lock.

The transfer stays together because it doesn't suspend. If a method checks a balance, suspends at `await`, and then tries to debit it, another operation can change the balance in between. Keep the check and update together, or recheck after resuming.

## Choose a lock for short, synchronous work

Suppose a synchronous callback needs to count each event before it returns. A [`Mutex`](mutex.md) protects the update without forcing the callback to be `async` or start a task.

```swift
import Synchronization

final class EventCounter: Sendable {
    private let count = Mutex(0)

    func recordEvent() {
        count.withLock { $0 += 1 }
    }

    func total() -> Int {
        count.withLock { $0 }
    }
}
```

Both methods can be called without `await`, and `total()` returns a copy. Keep the locked work short: a caller waiting for the mutex blocks its thread. With a manually managed lock like `NSLock`, every access must use the right lock, and taking the same lock again in a nested call can [deadlock](legacy-locking.md).

**What I want to remember:** An actor gives shared state a compiler-checked isolation boundary. A mutex is useful when the operation is short and the API needs to stay synchronous.

[Back to Swift Concurrency](index.md)
