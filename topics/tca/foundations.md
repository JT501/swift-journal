---
title: "TCA Foundations"
---

Suppose a counter screen can increment a number and reset it. TCA separates what the feature knows from what can happen to it:

| Piece | Responsibility |
|---|---|
| `State` | The feature's current data |
| `Action` | User events and results arriving from effects |
| Reducer | Synchronously update state and return effects |
| `Store` | Hold state, process actions, and run the returned effects |

## A small feature

This example uses the [modern syntax described in the topic index](index.md#version-context).

```swift
import ComposableArchitecture

@Reducer
struct Counter {
    @ObservableState
    struct State: Equatable {
        var count = 0
    }

    enum Action {
        case incrementButtonTapped

        case resetButtonTapped
    }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .incrementButtonTapped:
                state.count += 1
                return .none

            case .resetButtonTapped:
                state.count = 0
                return .none
            }
        }
    }
}
```

The reducer receives temporary `inout` access to state. `.none` means this action doesn't start any effects. `State` is `Equatable` so tests can compare before and after; `Action` doesn't need `Equatable` just to be sent.

## Do State, Action, and logic belong together?

Yes. A common TCA layout puts the feature's nested `State` and `Action` types, dependencies, and reducer `body` together in one feature type and file, such as `Counter.swift`. They describe one unit of behavior, and nesting gives them names such as `Counter.State` and `Counter.Action`.

The `Reduce` closure belongs inside that body's implementation; it isn't another model to put in a separate file. The SwiftUI view commonly lives in `CounterView.swift`, and tests in `CounterTests.swift`. As a feature grows, extract independently meaningful child features rather than automatically splitting every state and action type into separate files.

Name actions after events, such as a button tap, rather than everything their implementation happens to do. A tap might later reset a field, log analytics, and start a request without needing a new name.

## Who makes the feature run?

The reducer describes behavior; a store makes it happen. Using the `Counter` above:

```swift
@MainActor
func makeCounter() -> StoreOf<Counter> {
    let store = Store(initialState: Counter.State()) {
        Counter()
    }

    store.send(.incrementButtonTapped)
    return store
}
```

A view reads the state it needs and sends actions for user events. Business decisions belong in the reducer, not in button closures. `@ObservableState` lets the view observe the store properties it reads.

```swift
import SwiftUI

struct CounterView: View {
    // Received from the parent; this view doesn't create a new store.
    let store: StoreOf<Counter>

    var body: some View {
        HStack {
            Text("Count: \(store.count)")

            Button("Increment") {
                store.send(.incrementButtonTapped)
            }

            Button("Reset") {
                store.send(.resetButtonTapped)
            }
        }
    }
}
```

Reading `store.count` during `body` establishes an observation dependency. It doesn't require comparing the entire state for equality. See [Observation and Identity](observation-and-identity.md) for conditional reads and child-state replacement.

## Why value types?

A state snapshot can be compared with the next state to see exactly what changed. That powers reducer debugging with `._printChanges()` and exhaustive assertions in a `TestStore`. Hiding shared mutable reference objects inside state weakens that snapshot model.

Not every view needs its own reducer. A read-only document preview can simply receive a title and body text and render them. Add a feature when there is stateful behavior to model, not just because there is another view file.

**What I want to remember:** State describes now, actions describe events, the reducer makes decisions, and the store runs the feature.

[Next: Effects and Cancellation](effects-and-cancellation.md) · [Back to TCA](index.md)
