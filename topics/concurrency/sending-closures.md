---
title: "Sending Closures: The Captures Go Too"
---

A closure is a value, and it can hold references to other values through its captures. [Sending a value](sending-values.md) transfers its region; sending a closure must account for the objects it captures too.

## Hand off a closure and its document

```swift
final class Document {
    var text = ""
}

func runInTask(_ operation: sending @escaping () -> Void) {
    Task.detached {
        operation()
    }
}

func startReview() {
    let document = Document()
    document.text = "Draft"

    let review = {
        document.text += " — reviewed"
        print(document.text)
    }

    // `review` and its captured document belong to the same region.
    runInTask(review)

    // Either line would make the handoff unsafe:
    // print(document.text)
    // review()
}
```

The closure can capture a non-`Sendable` document because this is a handoff, not permission for both contexts to use it. `@escaping` lets the operation outlive `runInTask`; `sending` requires a safe transfer.

**`sending` doesn't mean “callable only once.”** The receiving code could call the closure repeatedly in its new isolation domain. What matters is preventing unsafe access from the old domain.

## How is that different from @Sendable?

`@Sendable` describes a closure whose captures satisfy safe cross-isolation use. For example, capturing a plain mutable, non-`Sendable` document in an `@Sendable` closure is rejected.

A `sending` parameter doesn't require that closure type. Instead, it allows a disconnected closure and its captured region to be handed over. Neither keyword makes unsynchronized concurrent mutations safe.

## Why do nested closures matter?

If an outer closure captures a document, and an inner closure tries to send it to a child task, their connected regions can make the transfer fail. Swift can't generally assume the outer closure is callable only once.

When the document is only needed by the child, create it inside the task-group body:

```swift
func reviewInGroup() async {
    await withTaskGroup(of: Void.self) { group in
        // Fresh region: this document isn't captured from outside the body.
        let document = Document()
        document.text = "Draft"

        group.addTask {
            document.text += " — reviewed"
            print(document.text)
        }

        // Don't read the document here or send it to a second child task.
    }
}
```

This avoids the outer capture rather than suppressing region checking with `nonisolated(unsafe)`.

## Three places to put sending

| Position | What is handed off? |
|---|---|
| `operation: sending @escaping () -> Void` | The closure value, including its connected captures |
| A closure parameter such as `(sending Document) -> Void` | A document passed into the closure |
| A closure result such as `() -> sending Document` | A disconnected document returned by the closure |

These are separate contracts. Sending a closure doesn't automatically make its parameters or result `sending`.

## Reading Mutex.withLock

Ignoring throwing and noncopyable-type details, the important shape of its signature is:

```text
withLock<Result>(
    _ body: (inout sending Value) -> sending Result
) -> sending Result
```

The closure itself isn't a `sending` parameter here. Instead, it receives temporary access to the protected value through `inout sending`: the updated state must be safe to hand back to the mutex when the closure finishes. The closure's `sending` result must separately be safe to hand to the caller.

That permits returning a non-`Sendable` result, but shouldn't permit returning an alias of mutable state still held by the mutex. The [compiler bug in our Mutex note](mutex.md#minimal-compiler-repro) is a hole in enforcement, not permission to share that state.

**What I want to remember:** Look at where `sending` appears. Sending the closure, sending an argument into it, and sending a result out of it are different boundaries.

[Back to Swift Concurrency](index.md)
