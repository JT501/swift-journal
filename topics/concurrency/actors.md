---
title: "Actors: Where `await` Belongs"
---

A bank needs to debit one account and credit another without letting a second operation see the transfer halfway through. If one actor owns both balances, it can do the whole update in a synchronous method without taking a lock explicitly.

```swift
actor Bank {
    private var balances = ["checking": 100, "savings": 100]

    func transfer(_ amount: Int, from source: String, to destination: String) -> Bool {
        guard amount > 0,
              source != destination,
              let sourceBalance = balances[source],
              let destinationBalance = balances[destination],
              sourceBalance >= amount,
              destinationBalance <= Int.max - amount
        else {
            return false
        }

        balances[source] = sourceBalance - amount
        balances[destination] = destinationBalance + amount
        return true
    }

    func balance(for account: String) -> Int? {
        balances[account]
    }
}

func example() async {
    let bank = Bank()

    if await bank.transfer(50, from: "checking", to: "savings"),
       let savings = await bank.balance(for: "savings") {
        print(savings) // 150
    }
}
```

- **Inside the actor:** `transfer` has no suspension point, so another operation on the same actor can't run between the debit and the credit.
- **Outside the actor:** The caller uses `await` because it may have to wait to enter the actor's isolation domain, even though `transfer` isn't declared `async`.
- **Across the boundary:** `balance(for:)` returns a snapshot, not access to the bank's mutable state. The transfer and the balance read are still two separate calls; if other tasks share the bank, they can make changes between them.

**What I want to remember:** Put steps that must happen together in one synchronous actor method. Use `await` at the boundary, not between those steps.

See [Actors vs. Locks: When to Use Each](actors-vs-locks.md) for the tradeoffs and a synchronous counter example.

[Back to Swift Concurrency](index.md)
