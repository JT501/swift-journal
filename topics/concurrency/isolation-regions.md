---
title: "Region-Based Isolation: Safe Transfer Without Sendable"
---

`Sendable` describes values that are safe to pass across isolation boundaries. But a mutable, non-`Sendable` object can sometimes cross a boundary too: Swift can prove that the transfer doesn't leave another context able to use it unsafely.

Region-based isolation lets the compiler reason about that specific use without requiring the whole type to conform to `Sendable`.

## What is a region?

A region groups non-`Sendable` values that might affect each other. Two references to the same mutable object belong to the same region. The compiler also merges regions conservatively when values might become connected through function calls or closure captures.

A region is a compile-time model, not a thread, queue, or runtime lock.

| Region | Meaning |
|---|---|
| Disconnected | Not tied to an actor or task's isolation; potentially safe to transfer |
| Actor-isolated | Associated with a particular actor's isolation |
| Task-isolated | Associated with a particular task's isolation |
| Invalid | Control-flow analysis couldn't reconcile the region's isolation; later access is rejected |

## Example: hand off a document, not a copy

An inbox actor stores mutable documents. A caller can prepare a fresh document and hand it over, as long as it doesn't keep using the document afterward.

```swift
final class Document {
    var text = ""
}

actor Inbox {
    private var documents: [Document] = []

    func accept(_ document: Document) {
        documents.append(document)
    }

    func latestText() -> String? {
        documents.last?.text
    }
}

func handOff(to inbox: Inbox) async {
    let document = Document()
    let alias = document // Same object, so both references share a region.
    alias.text = "Ready to review"

    // Transfer the document's region into the inbox actor's isolation.
    // No copy of the document is made.
    await inbox.accept(document)

    // Either line below would make the transfer unsafe and fail to compile:
    // document.text += " More edits"
    // alias.text += " More edits"

    // Read through the actor instead. String is a Sendable value snapshot.
    let text = await inbox.latestText()
    print(text as Any)
}
```

The local references haven't disappeared at runtime. Swift restricts their use after the transfer so the caller can't access the same mutable object outside the inbox's isolation. Waiting with `await` doesn't return that access to the caller: the inbox still holds the document after `accept` finishes.

## Example: hand off to a task

The same reasoning applies when a fresh document is captured by a new task. Using the `Document` class above:

```swift
func handOffToTask() {
    let document = Document()
    document.text = "Ready to review"

    // Transfer the document's region to the detached task.
    Task.detached {
        document.text += " — reviewed"
        print(document.text)
    }

    // Uncommenting this read makes the transfer unsafe:
    // print(document.text)
}
```

The task can mutate the document because the caller stops accessing it after the handoff. `Task.detached` doesn't inherit an actor's isolation, so this example makes the transfer to another task explicit. It isn't a recommendation to use detached tasks for every handoff.

Even a read afterward can conflict with the task's mutation. Declaring `document` with `let` only fixes the reference; the object's `text` remains mutable.

**Local scope alone isn't enough.** Swift checks aliases, possible connections, and later uses—not just where the variable was declared. A fresh local object often starts in a disconnected region, but its subsequent uses determine whether transferring it is safe.

## Why do aliases and merged regions matter?

Changing the variable name doesn't create a new object. Sending `document` and then editing `alias` would still access the inbox's document without its protection.

Even separately created objects can end up in one region. An ordinary function receiving two non-`Sendable` objects might connect them, so Swift conservatively merges their regions. A closure capturing both can connect them too. Transferring one value then affects what the caller can do with the other values in that region.

## How does control flow affect access?

If one branch transfers a document to one inbox and another branch transfers it to a different inbox, later code can't assume the document is still safe to access locally. Swift analyzes the possible paths, not just the branch that happens to run. Incompatible isolation assignments can produce an invalid region.

Using the same `Document` and `Inbox` types:

```swift
func routeDocument(to first: Inbox, or second: Inbox, useFirst: Bool) async {
    let document = Document()
    document.text = "Ready to route"

    if useFirst {
        // This path transfers the region to the first inbox.
        await first.accept(document)
    } else {
        // This path transfers it to the second inbox instead.
        await second.accept(document)
    }

    // The paths join here with incompatible actor-isolation assignments.
    // Uncommenting this line makes the transfer fail concurrency checking:
    // print(document.text)

    // Access through an actor remains safe: it protects its own state.
    let latestInFirst = await first.latestText()
    print(latestInFirst as Any)
}
```

Only one branch runs, and either handoff is safe on its own. The problem would be using the local reference afterward: the chosen inbox still holds the object. Reading through `first` is safe, but it doesn't imply the document went there—if `useFirst` is false, it went to `second`.

**What I want to remember:** A non-`Sendable` object isn't automatically forbidden from crossing isolation boundaries. What matters is whether Swift can prove the handoff is safe—including every alias and related value in its region.

[Back to Swift Concurrency](index.md)
