---
title: "Sending Values: Making a Handoff Explicit"
---

[Region-based isolation](isolation-regions.md) can prove that a non-`Sendable` object is safe to transfer. The `sending` keyword lets an API explicitly require that safe handoff for a parameter or promise it for a result.

## Sending a parameter

A non-`Sendable` `sending` argument must arrive in a disconnected region. The function can then transfer it elsewhere without leaving the caller able to access the same object unsafely.

```swift
final class Document {
    var text = ""
}

actor Inbox {
    private var documents: [Document] = []

    func accept(_ document: Document) {
        documents.append(document)
    }
}

func deliver(_ document: sending Document, to inbox: Inbox) async {
    // The parameter arrives disconnected, so we can transfer it to the actor.
    await inbox.accept(document)
}

func example() async {
    let inbox = Inbox()
    let document = Document()
    let alias = document
    alias.text = "Ready to review"

    await deliver(document, to: inbox)

    // Either access would make the handoff unsafe:
    // print(document.text)
    // alias.text += " More edits"
}
```

The contract applies to the whole region, including `alias`. `sending` doesn't copy the document, and the end of the call doesn't automatically give the caller permission to use it again.

### Why put sending on the helper?

Without a `sending` parameter, `deliver` can't assume its caller has given up access to the document. The caller might still hold aliases, making forwarding it to another isolation domain unsafe.

`sending` puts that requirement at the API boundary: Swift checks the caller's handoff, and the helper can rely on receiving a disconnected region. It isn't permission to skip checking; it tells Swift what to check on each side of the call.

## Sending a result

A `sending` result must be safe to return in its own disconnected region. For example, an actor can create a fresh document for a caller without keeping a reference to it:

```swift
actor DocumentFactory {
    func makeDocument() -> sending Document {
        let document = Document()
        document.text = "New draft"
        return document // Fresh object, not shared with the factory's state.
    }
}

func createAndDeliver(to inbox: Inbox) async {
    let factory = DocumentFactory()
    let document = await factory.makeDocument()

    // The returned document is disconnected, so it can be transferred again.
    await deliver(document, to: inbox)
}
```

Unlike an ordinary result that may be connected to the function's inputs, a `sending` result promises a disconnected handoff. The compiler checks that the implementation can satisfy that promise.

## Why isn't this the same as Sendable?

`Sendable` describes a type whose values are safe to pass across isolation boundaries. `sending` describes a particular transfer, allowing even a non-`Sendable` object to cross when the compiler can rule out unsafe sharing. It isn't a `Sendable` conformance or a general guarantee against logical race conditions.

## Caution: returning an object removed from actor storage

Suppose an actor looks up a stored document, removes it from its dictionary, and tries to return it as `sending Document`. Swift may still consider that document connected to the actor's region. Deleting the dictionary entry doesn't by itself prove that no other references remain.

`nonisolated(unsafe)` can bypass this checking, but it doesn't disconnect references at runtime or make the transfer safe. That requires a documented invariant: no aliases remain that could access the transferred object from the actor or any other isolation domain. Prefer a checked design; don't add the annotation just to silence the diagnostic.

### Example: take a privately owned document

Using the `Document` class above, this store creates its own document and never exposes it except through `take()`:

```swift
actor DocumentStore {
    private var stored: Document? = Document()

    func take() -> sending Document? {
        // SAFETY: `stored` is the only retained reference. Nothing else
        // exposes the document, and there are no callbacks holding aliases.
        // This annotation bypasses region checking for the local binding.
        nonisolated(unsafe) let document = stored

        // Remove the actor's reference before returning, with no suspension.
        stored = nil
        return document
    }
}

func takeAndEdit(from store: DocumentStore) async {
    if let document = await store.take() {
        // The store no longer retains this document.
        document.text += " — edited by the caller"
    }
}
```

The annotation tells the compiler to treat the local binding as disconnected; **clearing `stored` is what removes the actor's reference at runtime**. A second `take()` returns `nil`. If the store later adds another property or callback retaining the document, this safety argument must be revisited. `sending` on the result doesn't repair a false `nonisolated(unsafe)` assertion.

**What I want to remember:** Share safely with `Sendable`; hand off a disconnected non-`Sendable` value with `sending`. Neither keyword means “make a copy.”

[Back to Swift Concurrency](index.md)
