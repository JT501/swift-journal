---
title: "TCA Collections: Structure Versus Row Contents"
---

A parent needs to know which rows exist and their order. A row view needs its own contents. Those are different observation dependencies.

## Compose identified child features

Using [Counter and CounterView](foundations.md):

`CounterRow` is a wrapper that gives each list row its own stable ID while reusing `Counter` unchanged. It is optional: if `Counter.State` already had a stable `id` and conformed to `Identifiable`, the collection could be `IdentifiedArrayOf<Counter.State>` and `.forEach` could compose `Counter()` directly. Here the counter is reusable and has no row identity of its own, so the wrapper keeps list identity at the list's boundary.

```swift
import ComposableArchitecture
import Foundation
import SwiftUI

@Reducer
struct CounterRow {
    @ObservableState
    struct State: Equatable, Identifiable {
        let id: UUID
        var counter = Counter.State()
    }

    enum Action {
        case counter(Counter.Action)
    }

    var body: some ReducerOf<Self> {
        Scope(state: \.counter, action: \.counter) {
            Counter()
        }
    }
}

@Reducer
struct CounterList {
    @ObservableState
    struct State: Equatable {
        var rows: IdentifiedArrayOf<CounterRow.State> = []
    }

    enum Action {
        case addButtonTapped

        case removeButtonTapped(UUID)

        case rows(IdentifiedActionOf<CounterRow>)
    }

    @Dependency(\.uuid) var uuid

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .addButtonTapped:
                state.rows.append(CounterRow.State(id: uuid()))
                return .none

            case let .removeButtonTapped(id):
                state.rows.remove(id: id)
                return .none

            case .rows:
                return .none
            }
        }
        .forEach(\.rows, action: \.rows) {
            CounterRow()
        }
    }
}

struct CounterListView: View {
    let store: StoreOf<CounterList>

    var body: some View {
        List {
            Button("Add counter") {
                store.send(.addButtonTapped)
            }

            // Stable domain IDs, not offsets or mutable count values.
            ForEach(store.scope(\.rows, action: \.rows), id: \.state.id) { rowStore in
                HStack {
                    CounterView(store: rowStore.scope(\.counter, action: \.counter))

                    Button("Remove") {
                        store.send(.removeButtonTapped(rowStore.state.id))
                    }
                }
                .buttonStyle(.borderless)
            }
        }
    }
}
```

The `id: \.state.id` argument is optional. A scoped row `Store` is itself identifiable by object identity, so `ForEach(store.scope(\.rows, action: \.rows))` also works. Pass the state ID explicitly when SwiftUI should identify the row by its domain identity—for example, when that identity is tied to selection. The `IdentifiedArray` ID routes child actions either way; the `ForEach` argument chooses SwiftUI's view identity.

Three separate pieces cooperate:

- `IdentifiedArrayOf` stores ordered row state with stable IDs.
- Reducer `.forEach` routes identified actions and runs child behavior.
- SwiftUI `ForEach` renders scoped stores; it doesn't run the reducers itself.

## Which change should affect which view?

Changing a row's count should update that counter view without the parent acquiring a count dependency merely by listing rows. Adding, removing, or reordering rows changes collection structure.

If the parent additionally reads `store.rows.first?.counter.count` to display a summary, it deliberately observes the first row's count. A filter that reads every row's contents can also introduce wider dependencies. Observation isn't a promise that parents never update when a child changes.

The scoped result is a collection, so presentation code can use operations such as `prefix(3)`. Slicing what is displayed doesn't remove those rows from reducer state or cancel their effects. Keep stable identities after slicing or enumeration; an offset is a display position, not an entity ID.

Avoid manually creating a row store by force-unwrapping a lookup such as `store.rows[id: id]!`: that row may already have been removed. Use the scoped collection with `ForEach` as above; TCA routes actions by ID and manages the child stores. If you need a one-off lookup, handle the optional result in case the row no longer exists.

**What I want to remember:** Observe collection structure in the parent and row contents where they're displayed.

[Back to TCA](index.md)
