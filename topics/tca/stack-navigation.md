---
title: "TCA Stack Navigation and Feature Communication"
---

A sheet is a child nested under its presenter. A navigation stack is an ordered collection of active destinations. Put application-level stack ownership in a composition root rather than making a reusable list know every possible screen.

## Model the destinations and path

Using [Counter](foundations.md) as a small destination:

```swift
import ComposableArchitecture

@Reducer
enum AppPath {
    case counter(Counter)
}

extension AppPath.State: Equatable {}

@Reducer
struct NavigationDemo {
    @ObservableState
    struct State: Equatable {
        var path = StackState<AppPath.State>()
    }

    enum Action {
        case openCounterButtonTapped

        case path(StackActionOf<AppPath>)
    }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .openCounterButtonTapped:
                state.path.append(.counter(Counter.State()))
                return .none

            case .path:
                return .none
            }
        }
        .forEach(\.path, action: \.path) {
            AppPath.body
        }
    }
}
```

The path enum can grow to include other destinations, such as a document editor. `.forEach` routes element actions and manages their lifetimes. The view binds a native `NavigationStack` to the scoped path store.

```swift
import SwiftUI

struct NavigationDemoView: View {
    @Bindable var store: StoreOf<NavigationDemo>

    var body: some View {
        NavigationStack(path: $store.scope(\.path, action: \.path)) {
            Button("Open counter") {
                store.send(.openCounterButtonTapped)
            }
            .navigationTitle("Counters")
        } destination: { pathStore in
            // Each case carries a scoped child store, not just raw child state.
            switch pathStore.case {
            case let .counter(counterStore):
                CounterView(store: counterStore)
                    .navigationTitle("Counter")
            }
        }
    }
}
```

The exhaustive switch makes new destination cases visible to the compiler. Native view APIs don't eliminate `StackState`, identified path actions, or reducer composition.

## Stack identity is not a row index

Stack actions describe pushing a destination, sending an action to an element ID, or popping from an element ID. Stack element IDs are not model UUIDs or array offsets. Tests may spell them with integer literals, but a new push after a pop doesn't mean an old ID is reused.

An initial `StackState` can describe a deep link with several destinations already active, including a destination that itself presents a sheet.

## Why didn't editing update the root list?

If a detail feature receives a value copy of a record, local edits don't automatically update the root collection. That's useful isolation, not a reason to switch to shared mutable references without considering ownership.

A useful communication pattern is a delegate action:

```text
Detail commits an edit
  → Detail sends .delegate(.recordUpdated(updatedRecord))
  → App handles the event and updates its record by stable ID
```

The parent handles the business result, rather than depending on the child's exact Save button implementation. A reducer `onChange` hook can publish a delegate event whenever the committed record changes. Changes to a scratch draft shouldn't publish a commit.

The delegate is an effect-emitted action, so an exhaustive integration test observes it separately from the Save action. Don't assert that the root changed before the delegate has been processed.

## Coordinating siblings

A document editor and a document list can remain independent child features. When the editor saves, it sends the updated document to the root feature, which updates its list by document ID. The editor doesn't need to know how the list is stored.

If creating the document needs an ID or timestamp, use UUID and date dependencies so the behavior is testable. If coordination assumes a list screen precedes the editor in the path, validate that assumption instead of silently dropping a result. A navigation stack permits many sequences; its type doesn't guarantee every business rule.

**What I want to remember:** The composition root owns navigation and integration policy; children report meaningful results.

[Next: Domain Modeling](domain-modeling.md) · [Back to TCA](index.md)
