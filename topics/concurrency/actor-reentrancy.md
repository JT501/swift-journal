---
title: "Actor Reentrancy: What Can Change Across await?"
---

An actor runs one job at a time. **Reentrancy** means that when an actor method suspends, another call can run on that same actor before the first call resumes. The two calls take turns; they don't access the actor's state simultaneously. This lets the actor keep serving requests while one call waits on slow work.

## How can two withdrawals overdraw an account?

Suppose the balance is 100. A withdrawal of 80 checks the balance, then suspends while waiting for approval. A second withdrawal of 50 completes, leaving 50. If the first withdrawal resumes and subtracts 80 based on its earlier check, the balance becomes **−30**.

This is a **race condition, not a data race**: both withdrawals accessed the balance one at a time. The mistake was assuming that the result of the first check would still hold after `await`.

## Where should the balance check go?

Wait for approval while the balance is still consistent. Then check the **current** balance and debit it without a suspension point between those steps:

```swift
actor Account {
    private var balance = 100

    func withdraw(_ amount: Int, approve: @Sendable (Int) async -> Bool) async -> Bool {
        guard amount > 0 else { return false }
        guard await approve(amount) else { return false }

        guard balance >= amount else { return false }
        balance -= amount
        return true
    }
}
```

The amount is a local constant, so checking that it's positive before `await` is fine. The balance is mutable actor state, so its check belongs **after** `await`. If the balance is now 50, the withdrawal of 80 is declined.

## Reducing suspension points by passing isolation

Suppose each account is also an actor. If a bank transfer needs `await` to debit one account and another `await` to credit the other, another job can observe the transfer halfway through. When accounts are meant to run on their bank's executor, the bank can **pass itself as the isolation source** instead:

```swift
actor Account {
    let isolation: Bank
    var balance = 100

    init(isolation: Bank) {
        self.isolation = isolation
    }

    nonisolated var unownedExecutor: UnownedSerialExecutor {
        isolation.unownedExecutor
    }
}

actor Bank {
    func openAccount() -> Account {
        Account(isolation: self)
    }

    func transferOne(from source: Account, to destination: Account) -> Bool {
        guard source.isolation === self, destination.isolation === self else { return false }

        let debited = source.assumeIsolated { account in
            guard account.balance > 0 else { return false }

            account.balance -= 1
            return true
        }
        guard debited else { return false }

        destination.assumeIsolated { $0.balance += 1 }
        return true
    }
}
```

There are two distinct steps here. Forwarding `unownedExecutor` makes the accounts run on the bank's serial executor. `assumeIsolated` then **checks** that execution is already on the account's executor and gives the closure synchronous access to the account. It doesn't switch executors; a wrong assumption traps. That's why the transfer checks that both accounts belong to this bank.

A caller outside the bank still writes `await bank.transferOne(from: source, to: destination)`, but the debit and credit inside it need **no suspension point**. No job on that executor can observe the half-finished transfer. This is useful when actors deliberately share an executor; it doesn't eliminate a genuine wait such as fraud approval.

[Back to Swift Concurrency](index.md)
