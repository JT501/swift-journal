---
title: "Actor Enqueuing: One Hop, Several Operations"
---

Suppose a bank actor owns both a checking balance and a savings balance. After transferring money, you want to read the resulting savings balance before another task changes it.

```swift
actor Bank {
    private var checking = 100
    private var savings = 100

    func transferToSavings() {
        guard checking >= 50 else { return }

        checking -= 50
        savings += 50
    }

    func savingsBalance() -> Int {
        savings
    }
}

extension Bank {
    func run<Result: Sendable>(_ body: @Sendable (isolated Bank) -> Result) -> Result {
        body(self)
    }
}

func separateCalls(on bank: Bank) async -> Int {
    await bank.transferToSavings()
    return await bank.savingsBalance()
}

func batchedCalls(on bank: Bank) async -> Int {
    await bank.run { bank in
        bank.transferToSavings()
        return bank.savingsBalance()
    }
}
```

`run` is a helper we define, not a built-in actor method. It calls `body(self)` while isolated to the bank. The `isolated Bank` parameter gives the closure synchronous access to that bank, and `Result: Sendable` lets it return a value such as `Int` across the actor boundary.

With separate calls, another task can change the balance between the transfer and the read. With `run`, neither call inside the synchronous closure needs `await`, and another job on the bank can't interleave between them.

## Does every `await` enqueue a job?

No. `await` marks a possible suspension, not a guaranteed hop. Swift can sometimes avoid enqueuing a separate job for consecutive calls on the same actor. Group operations because they belong together, not because you counted the `await`s.

## Does the actor run jobs in submission order?

Not necessarily. An actor's serial executor runs one job at a time, but Swift doesn't guarantee first-in, first-out scheduling. If two steps must happen in a particular order, put them in the same synchronous operation or express the dependency explicitly.

[Back to Swift Concurrency](index.md)
