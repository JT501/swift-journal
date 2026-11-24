---
title: "Actor Performance: Measure the Work, Not Just the Hop"
---

An actor and a mutex can protect the same cache but spend different amounts of time coordinating access. Measuring one tiny lookup doesn't tell me which approach will work better for a whole app.

## Wall-clock time versus CPU time

- **Wall-clock time:** How long the caller waits for the operation to finish.
- **CPU time:** How much processor time the work consumes, potentially across several threads.

Under contention, a mutex can finish an operation sooner while using more CPU. Waiting for a lock blocks a thread; waiting to enter an actor can suspend the task and leave the thread available for other work. Neither metric alone tells the whole story, and results depend on the workload and toolchain.

## Batch work at the isolation boundary

Suppose a cache needs to remove every entry whose value exceeds a limit. Fetching keys, reading each value, and removing each matching entry through separate actor calls creates repeated opportunities for suspension. Another task could also change a value after the check but before the removal.

The example below shows **where to put the loop**: inside the actor, rather than in the caller with an `await` for every lookup and removal. `filter` checks all entries during one synchronous cleanup call. This demonstrates batching, not a benchmark.

```swift
actor Cache {
    private var values: [String: Int]

    init(values: [String: Int]) {
        self.values = values
    }

    func removeValues(above limit: Int) {
        values = values.filter { $0.value <= limit }
    }

    func snapshot() -> [String: Int] {
        values
    }
}

func example() async {
    let cache = Cache(values: ["small": 10, "large": 100])

    // One actor call handles every entry, with no await inside the cleanup.
    await cache.removeValues(above: 50)

    // Another task could change the cache before this separate read.
    let remaining = await cache.snapshot()
    print(remaining) // ["small": 10] if no other task changes the cache.
}
```

There is no suspension between checking an entry and deciding whether to keep it. The caller enters the actor for the cleanup once, rather than accessing it separately for each entry.

**The cleanup and snapshot are not one transaction.** Another job on the cache can run between the two calls. That's fine if I just want a later snapshot, but it's a correctness issue if I require the exact state immediately after my cleanup. This is possible interleaving, not a data race.

The [`run` helper](actor-enqueuing.md) offers another way to group synchronous operations when the grouping belongs at the call site.

```swift
extension Cache {
    func run<Result: Sendable>(_ body: @Sendable (isolated Cache) -> Result) -> Result {
        body(self)
    }
}

func cleanupAndSnapshot(on cache: Cache) async -> [String: Int] {
    await cache.run { cache in
        cache.removeValues(above: 50)
        return cache.snapshot()
    }
}
```

`run` is a helper we define, not a built-in actor method. Its `isolated Cache` parameter lets the closure call the cache's methods synchronously. Here the cleanup **and** snapshot happen during one actor access, so another job on the cache can't change the values between them. Only the outer call needs `await`; the returned dictionary is a value snapshot.

## What makes a useful comparison?

A mutex can also batch cleanup inside one critical section if its API supports it. Comparing a batched actor against thousands of separately locked calls measures an API-design difference, not a universal advantage of actors.

Compare equivalent work and correctness guarantees in an optimized build. Include realistic contention, wall-clock time, CPU time, and the effect on other work in the app. Keep batches bounded: a long synchronous actor operation delays other jobs on that actor, just as a long critical section delays other lock users.

**What I want to remember:** Choose a correct isolation boundary first, then measure realistic workloads. Fewer synchronization boundaries can help, but neither actors nor locks are always faster.

[Back to Swift Concurrency](index.md)
