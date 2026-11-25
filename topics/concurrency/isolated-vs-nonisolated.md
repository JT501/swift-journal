---
title: "Isolated, nonisolated, and nonisolated(nonsending)"
---

Isolation is about permission to access protected state, not choosing a thread. These three tools answer different questions about where that permission comes from.

| Declaration | Meaning |
|---|---|
| `account: isolated BankAccount` | Run within this specific account's isolation |
| `nonisolated` | Don't require the enclosing actor's isolation |
| `nonisolated(nonsending)` | Let an async function inherit the caller's isolation |

## Isolated: use a specific actor's isolation

An actor's instance methods are actor-isolated by default. A helper outside the actor can also receive an **isolated actor parameter**:

```swift
actor BankAccount {
    let holder: String
    private(set) var balance = 100

    init(holder: String) {
        self.holder = holder
    }

    func withdraw(_ amount: Int) -> Bool {
        guard amount > 0, balance >= amount else { return false }

        balance -= amount
        return true
    }

    nonisolated var details: String {
        "Account holder: \(holder)"
    }
}

// `isolated` makes this function run within the supplied account's isolation.
// It can access that account synchronously; outside callers await entry.
func withdrawAndRead(from account: isolated BankAccount) -> Int? {
    guard account.withdraw(50) else { return nil }

    return account.balance
}

func example() async {
    let account = BankAccount(holder: "Sam")
    let remaining = await withdrawAndRead(from: account)
    print(remaining as Any) // Optional(50)
    print(account.details) // No await needed.
}
```

`withdrawAndRead` runs within the supplied account's isolation. It doesn't need to be declared `async`: the outside caller awaits access, and the helper can withdraw and read synchronously. No job on that account can change the balance between those steps. This is the same principle used by an [isolated closure in the `run` helper](actor-enqueuing.md).

## Nonisolated: opt out of the enclosing actor's isolation

The `details` property above doesn't need access to mutable actor state. It uses only an immutable, `Sendable` string, so marking it `nonisolated` lets callers read it synchronously.

It couldn't read `balance`: that property is mutable and protected by the actor. `nonisolated` doesn't bypass that protection.

For an **async** function, plain `nonisolated` has settings-dependent behavior. Without caller-isolation inheritance enabled, a nonisolated async function switches to the global concurrent executor. With `NonisolatedNonsendingByDefault` enabled, it inherits the caller's isolation instead. Don't treat `nonisolated` as a synonym for “background.”

## Nonisolated(nonsending): inherit the async caller's isolation

Sometimes an actor owns a non-`Sendable` helper object that needs to do async work. Keeping the helper in the caller's isolation avoids unnecessarily sending it across a boundary.

```swift
final class Draft {
    private var text = ""

    nonisolated(nonsending)
    func appendAfterPause(_ suffix: String) async {
        await Task.yield()
        text += suffix
    }
}

actor Editor {
    private let draft = Draft()

    func edit() async {
        // Same actor isolation, but the call can still suspend at Task.yield().
        await draft.appendAfterPause("!")
    }
}
```

The draft stays within the editor's isolation during this call; it doesn't need a `Sendable` conformance. Here, `edit()` is isolated to this editor instance, and `appendAfterPause` inherits that isolation. If it suspends, other jobs on the editor can run; when it resumes, it runs under the same editor's isolation again.

The method inherits isolation dynamically: a main-actor caller would supply main-actor isolation, while a caller without actor isolation wouldn't supply an actor at all.

**Why still use `await`?** Inheriting isolation avoids an executor hop, not the possibility of suspension. `appendAfterPause` is async and can suspend at `Task.yield()`, so its caller must acknowledge that with `await`, even though both run within the editor's isolation.

Removing `Task.yield()` alone doesn't remove the caller's need for `await`: the method is still declared `async`, even if its implementation never actually suspends. If it no longer needs async work, remove `async` and `nonisolated(nonsending)` from the method, then remove `await` at the call site.

This doesn't give `Draft` direct access to the editor's private state. It also doesn't remove suspension: while the call waits at `Task.yield()`, other work on the editor can run. The [reentrancy rules](actor-reentrancy.md) still apply.

Async helpers called from this method also need compatible isolation behavior. One `nonisolated(nonsending)` annotation doesn't automatically change an entire call chain.

## What about @concurrent?

On Swift 6.2 and later, `@concurrent` explicitly makes an async function run on the global concurrent executor rather than inheriting the caller's actor isolation. Use it when leaving that isolation is intentional. It doesn't promise a particular thread.

## Can a synchronous method use nonisolated(nonsending)?

No. Swift only permits `nonisolated(nonsending)` on async functions. It controls whether an async call inherits the caller's isolation instead of switching to the global concurrent executor.

For synchronous code, use `nonisolated` to opt out of the enclosing actor's isolation, or an `isolated` actor parameter when the function needs access to a specific actor's state.

## Can a function have two isolated actor parameters?

No. Swift permits at most one `isolated` actor parameter per function. Two actors can have independent isolation domains, and a function can't simply acquire synchronous access to both by marking both parameters `isolated`.

Other actor parameters can be ordinary, non-isolated parameters, but accessing their isolated state generally requires `await`. If related updates must stay synchronous, consider having one actor own the related state. Deliberately [sharing an executor](actor-reentrancy.md#reducing-suspension-points-by-passing-isolation) is a more advanced design; it still doesn't permit two `isolated` parameters.

**What I want to remember:** `isolated` selects an actor, `nonisolated` opts out of an enclosing actor, and `nonisolated(nonsending)` follows the async caller.

[Back to Swift Concurrency](index.md)
