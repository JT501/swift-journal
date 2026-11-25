---
title: "MainActor by Default: A Default, Not a Guarantee"
---

Default main-actor isolation is a **per-target setting**. It makes eligible declarations without explicit isolation main-actor-isolated, which can reduce annotations in UI code. It doesn't force every declaration or dependency onto the main actor.

## Which settings are involved?

| Setting | What it changes |
|---|---|
| `.defaultIsolation(MainActor.self)` in a SwiftPM target | The target's default isolation for eligible declarations |
| `NonisolatedNonsendingByDefault` upcoming feature | Nonisolated async functions inherit the caller's isolation by default |
| `InferIsolatedConformances` upcoming feature | Allows inference of isolated protocol conformances |

In Xcode, look for **Default Actor Isolation** and **Approachable Concurrency** in the target's build settings. These are distinct choices: inheriting an async caller's isolation doesn't automatically make that caller main-actor-isolated.

## Example: a UI model

Compile the examples in this note with default isolation set to `MainActor` and the two upcoming features above enabled.

```swift
// Inferred @MainActor in this target.
final class Model {
    private var count = 0

    func increment() {
        count += 1
    }
}

nonisolated func update(_ model: Model) async {
    // Opting out means this function doesn't start with main-actor access.
    await model.increment()

    // Calling model.increment() without await here would be rejected.
}
```

The model protects its count through main-actor isolation. `nonisolated` opts the helper out of that default; `await` lets it request access to the model instead.

## Protocols can change what's inferred

A dependency might define a protocol outside main-actor isolation. Here, `nonisolated` makes that explicit even inside our main-actor-default target:

```swift
nonisolated protocol ConcurrentOperation: Sendable {
    func operate() -> String
}

// Direct conformance to this Sendable protocol changes default inference:
// this type is inferred nonisolated, not @MainActor.
final class DirectModel: ConcurrentOperation {
    func operate() -> String {
        "No actor-owned state needed"
    }
}
```

The synchronous requirement must be usable without entering the main actor. This example doesn't mean every `Sendable` type is nonisolated: an explicitly `@MainActor` class can also be `Sendable`, but its actor-isolated methods still require main-actor access.

Moving the conformance to an extension doesn't apply the same inference to the type:

```swift
// This type keeps the target's default main-actor isolation.
final class ExtensionModel {
    private var count = 0

    func increment() {
        count += 1
    }

    // Explicitly nonisolated to satisfy the protocol's synchronous requirement.
    // Reading `count` here would be rejected.
    nonisolated func operate() -> String {
        "No actor-owned state needed"
    }
}

extension ExtensionModel: ConcurrentOperation {}
```

Without `nonisolated` on `operate`, the extension's conformance would produce an isolation diagnostic under these settings. If a synchronous requirement really needs main-actor state, don't remove its protection just to satisfy the protocol—consider a suitable isolated conformance or a different API.

**What I want to remember:** Check the target settings, explicit annotations, and protocol requirements together. Main-actor-by-default is a starting point for inference, not a guarantee about every declaration.

[Back to Swift Concurrency](index.md)
