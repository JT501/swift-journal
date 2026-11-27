---
title: "TCA Presentation and Draft Editing"
---

A presentation isn't just a Boolean. An edit sheet needs a draft, and dismissing it should end that draft's lifetime. Optional child state expresses both facts: `nil` means absent; a value means presented with that data.

## Edit a copy and commit deliberately

Using [EditForm](composition-and-bindings.md):

```swift
import ComposableArchitecture

@Reducer
struct DocumentEditor {
    @ObservableState
    struct State: Equatable {
        @Presents var edit: EditForm.State?
        var title = "Original"
    }

    enum Action {
        case cancelButtonTapped

        case edit(PresentationAction<EditForm.Action>)

        case editButtonTapped

        case saveButtonTapped
    }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .cancelButtonTapped:
                state.edit = nil
                return .none

            case .edit:
                return .none

            case .editButtonTapped:
                // A value copy: editing the draft won't change the saved title.
                state.edit = EditForm.State(title: state.title)
                return .none

            case .saveButtonTapped:
                guard let draft = state.edit else { return .none }

                state.title = draft.title
                state.edit = nil
                return .none
            }
        }
        .ifLet(\.$edit, action: \.edit) {
            EditForm()
        }
    }
}
```

`.ifLet` composes the optional feature and its presentation lifecycle. `PresentationAction` distinguishes child events (`.presented`) from dismissal (`.dismiss`). Presentation integration clears state for an interactive dismissal too.

The view chooses whether to show this state as a sheet, popover, or another presentation style. The form itself doesn't need to own the surrounding navigation stack, title, or Save/Cancel toolbar.

## Connect it to a native SwiftUI sheet

Using `DocumentEditor` above and [EditFormView](composition-and-bindings.md):

```swift
import SwiftUI

struct DocumentEditorView: View {
    @Bindable var store: StoreOf<DocumentEditor>

    var body: some View {
        VStack {
            Text(store.title)

            Button("Edit") {
                store.send(.editButtonTapped)
            }
        }
        .sheet(item: $store.scope(\.$edit, action: \.edit)) { editStore in
            NavigationStack {
                EditFormView(store: editStore)
                    .navigationTitle("Edit title")
                    .toolbar {
                        ToolbarItem(placement: .cancellationAction) {
                            Button("Cancel") {
                                store.send(.cancelButtonTapped)
                            }
                        }

                        ToolbarItem(placement: .confirmationAction) {
                            Button("Save") {
                                store.send(.saveButtonTapped)
                            }
                        }
                    }
            }
        }
    }
}
```

The projected `\.$edit` scope supplies the presentation binding. SwiftUI writing `nil` on dismissal becomes `.edit(.dismiss)`, which the presentation reducer integration processes. Creating the sheet still starts with an action that populates state.

Why not `$store.edit`? That is a `Binding<EditForm.State?>` — raw child state with no actions. The sheet body needs a `StoreOf<EditForm>` to send child actions, and `.sheet(item:)` needs an `Identifiable` item, which the scoped presentation store provides. A direct state binding would also bypass the child reducer and its `BindingReducer`.

For enum destinations in TCA 1.26.2, scope the projected destination and then choose its case, for example `$store.scope(\.$destination, action: \.destination).edit`. Some early observation examples use nonprojected optional-chained scopes; don't copy that historical spelling as the preferred current API.

Unwrapping an optional store in a view replaces the old `IfLetStore` rendering helper, not the reducer's `.ifLet` or presentation lifecycle handling.

## What should tests prove?

- Opening initializes the draft from the committed value.
- Typing changes the draft, not the committed value.
- Cancel discards changes.
- Save commits changes and clears presentation state.
- Interactive dismissal leaves no active draft.

You can initialize the store with `edit` already populated to preview a particular editing state. That's state-driven deep linking, not a script replaying taps. The same idea is possible in other architectures when navigation state is externally owned.

## Child-driven dismissal

A child can use `@Dependency(\.dismiss)` and call `await dismiss()` inside an effect when its own work is complete. This requests removal through the parent's presentation or stack integration.

If the parent needs a result first, the child sends a semantic delegate event with that result before requesting dismissal. Don't hide saving logic in a view's disappearance callback: a disappearance isn't necessarily a confirmed commit.

**What I want to remember:** Optional draft state ties presentation to its data; Save and Cancel have different state transitions.

[Next: Stack Navigation](stack-navigation.md) · [Back to TCA](index.md)
