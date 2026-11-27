---
title: "TCA Observation and Identity"
---

`@ObservableState` lets a view track the state properties it reads, so a change can invalidate views that depend on those properties. This is separate from reducer composition: `Scope`, `.ifLet`, and `.forEach` determine which child reducers handle actions.

## Reads establish dependencies

Suppose a parent can show or hide a summary of a child counter. The child still works while its summary is hidden.

Using [Counter and CounterView](foundations.md):

```swift
import ComposableArchitecture
import SwiftUI

@Reducer
struct ObservationDemo {
    @ObservableState
    struct State: Equatable {
        var child: Counter.State? = Counter.State()
        var isShowingSummary = false
    }

    enum Action {
        case child(Counter.Action)

        case resetChildButtonTapped

        case toggleChildButtonTapped

        case toggleSummaryButtonTapped
    }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .child:
                return .none

            case .resetChildButtonTapped:
                // Reset the existing child to its initial values.
                state.child = Counter.State()
                return .none

            case .toggleChildButtonTapped:
                state.child = state.child == nil ? Counter.State() : nil
                return .none

            case .toggleSummaryButtonTapped:
                state.isShowingSummary.toggle()
                return .none
            }
        }
        .ifLet(\.child, action: \.child) {
            Counter()
        }
    }
}

struct ObservationDemoView: View {
    let store: StoreOf<ObservationDemo>

    var body: some View {
        VStack {
            Button("Toggle summary") {
                store.send(.toggleSummaryButtonTapped)
            }

            // When the flag is false, the count read doesn't execute.
            if store.isShowingSummary, let count = store.child?.count {
                Text("Summary: \(count)")
            }

            // Observe child presence/identity, not every field in the child.
            if let childStore = store.scope(\.child, action: \.child) {
                CounterView(store: childStore)
            }

            Button("Toggle child") {
                store.send(.toggleChildButtonTapped)
            }

            Button("Reset child") {
                store.send(.resetChildButtonTapped)
            }
        }
    }
}
```

When the summary is hidden, the parent doesn't acquire a dependency on the child's count through that summary. The child view still reads the count and updates for it. Showing the summary adds that read during the next parent evaluation.

This isn't a guarantee that SwiftUI never evaluates the parent for other reasons. It describes which state changes create dependencies through these reads.

## Presence, contents, and replacement are different

The parent's `if let` needs to react when the child appears, disappears, or is replaced. It doesn't need to react to every count edit merely to pass the child store along.

Whole-feature replacement also needs to keep observation connected to the current state. TCA handles that at the store and observable-state boundaries. This is why simply making one leaf property observable isn't equivalent to integrating observation throughout the feature tree. Don't rely on every fresh assignment to generate a notification: equality-based optimizations can suppress equal values.

The view's `if let` renders the child. The reducer's `.ifLet` runs its behavior and manages its lifetime. Removing an old view helper doesn't remove that composition requirement.

## Which identity are we talking about?

| Identity | Purpose |
|---|---|
| Domain ID, such as a document's UUID | Identify the same business entity across edits and reordering |
| Scoped store identity | Give SwiftUI a stable reference for a particular active child |
| Internal observation bookkeeping | Keep observation connected as TCA state changes; this is managed by TCA |

Don't add a random model ID or a SwiftUI `.id(UUID())` to “fix” observation. Internal observation identity is library machinery, not an app-level persistence key.

For example, an editor can switch from one document to another while remaining the same kind of child feature:

```swift
struct Document: Identifiable {
    let id: UUID
    var title: String
}

struct EditorState {
    let documentID: UUID
    var draftTitle: String
}

var editor: EditorState?

// Opening a different document replaces the editor's state.
editor = EditorState(documentID: document.id, draftTitle: document.title)
```

The `documentID` stays the same while editing that document's title. When opening another document, use that document's existing ID. This is domain identity; it answers “which document?” It is different from resetting a child to its initial values, as in the counter example above.

For enums, changing the active case is structural even if the associated child data came from the same value. For collections, membership and order differ from individual row contents; see [Observing Collections](observing-collections.md).

## Value semantics still have boundaries

TCA supports observation of value state inside a store. That doesn't promise that arbitrary standalone copies of an observable struct can be independently observed, replaced, and reassigned with perfect notification. Avoid custom registrar code or manual manipulation of underscored identity fields.

**What I want to remember:** Observation follows reads, structural identity tracks replacement, and reducers still own behavior.

[Back to TCA](index.md)
