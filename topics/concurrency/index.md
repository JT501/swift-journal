---
title: Swift Concurrency
---

1. [Swift Concurrency Foundations](foundations.md) — four questions about sharing mutable state safely.
2. [Legacy Locking](legacy-locking.md) — `NSLock`, deadlocks, and why recursive locks deserve a closer look.
3. [Keeping State Inside the Lock](keeping-state-inside-the-lock.md) — `OSAllocatedUnfairLock`, `Sendable`, and why a locked lookup can still leak mutable state.
4. [Mutex: Sharing Versus Transferring State](mutex.md) — non-`Sendable` state, `sending`, and a compiler bug to watch for.
5. [Actors: Where `await` Belongs](actors.md) — synchronous work inside an actor and asynchronous access from outside.
6. [Actor Enqueuing: One Hop, Several Operations](actor-enqueuing.md) — batch synchronous actor work and understand what `await` says about scheduling.
7. [Actors vs. Locks: When to Use Each](actors-vs-locks.md) — choose based on how callers need to access shared state.
8. [Actor Reentrancy: What Can Change Across `await`?](actor-reentrancy.md) — avoid acting on stale state after a suspension.
9. [Actor Performance: Measure the Work, Not Just the Hop](actor-performance.md) — batch related work and compare equivalent workloads.
10. [Isolated, nonisolated, and nonisolated(nonsending)](isolated-vs-nonisolated.md) — where isolation comes from and when an async function inherits it.

[Back to the learning path](../../index.md)
