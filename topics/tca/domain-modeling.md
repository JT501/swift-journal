---
title: "TCA Domain Modeling: Make Invalid States Hard to Express"
---

Suppose a detail screen can show either an edit sheet or a delete-confirmation alert, but not both. Two optional properties allow four presence/absence combinations; one of those combinations is invalid.

With four mutually exclusive destinations, separate optionals allow 16 combinations, but only five are meaningful: none, or exactly one destination.

## Use an optional enum for exclusive destinations

Using [EditForm](composition-and-bindings.md), modern TCA can describe the destination reducer as:

```swift
import ComposableArchitecture

@Reducer
enum DetailDestination {
    case alert(AlertState<Alert>)
    case edit(EditForm)

    @CasePathable
    enum Alert {
        case confirmDeletion
    }
}

extension DetailDestination.State: Equatable {}
```

The presenting feature then owns one `@Presents var destination: DetailDestination.State?`, one presentation action wrapping `DetailDestination.Action`, and one `.ifLet` composition. The macro generates state and action cases for the destinations.

`nil` means nothing is presented. A non-`nil` state chooses exactly one case and carries the data required by that destination. Adding another case doesn't introduce impossible combinations of simultaneous presentations.

An `AlertState` is a value description of an alert and its button actions, not a SwiftUI view stored in state. Tests can compare it meaningfully.

## Don't solve every problem with another reducer

A read-only document preview can receive a title and body text and display them. It doesn't need its own TCA feature just to show these values. If it later gets an Edit button or loads data, its parent can handle that behavior or you can introduce a dedicated feature.

```swift
import SwiftUI

struct DocumentPreview: View {
    let title: String
    let bodyText: String

    var body: some View {
        VStack(alignment: .leading) {
            Text(title).font(.headline)
            Text(bodyText)
        }
    }
}
```

The parent can render it with `DocumentPreview(title: document.title, bodyText: document.bodyText)`. There is no `Action` type here: this is an ordinary SwiftUI view, not a reducer. `Never` only matters when a generic API *requires* an action type but no action can occur; you don't need it for this example.

Similarly, derive remaining duration from elapsed time and total duration rather than storing all three independently. Keep enough state to describe the feature, not redundant values that can drift apart.

```swift
struct SessionTime {
    var elapsedSeconds = 0
    var totalSeconds = 60

    var remainingSeconds: Int {
        max(0, totalSeconds - elapsedSeconds)
    }
}
```

When `elapsedSeconds` changes, `remainingSeconds` follows automatically; there is no third stored value to keep in sync.

## Types don't enforce every business rule

A navigation path isn't invalid merely because it contains an editor. An editor can be a valid destination, including when opened directly from a deep link. The problem is missing required data: if editing needs a document ID, the editor's state should carry that ID. If the ID no longer identifies a document, decide whether to show an error, dismiss the editor, or load the document. Model and handle that case explicitly; an enum of destination types alone can't guarantee that referenced data exists.

Value-based editing also needs an explicit commit policy. A draft copy protects against accidental edits, but propagating the committed value back to its owner remains the parent's responsibility.

**What I want to remember:** Use structs for data that belongs together, enums for mutually exclusive choices, and runtime validation for invariants the types don't express.

[Next: Dependencies](dependencies.md) · [Back to TCA](index.md)
