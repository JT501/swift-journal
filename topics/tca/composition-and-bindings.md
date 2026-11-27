---
title: "TCA Composition and Bindings"
---

Composition means building a larger feature from smaller features with their own state, actions, and behavior. The parent decides how the pieces fit together; the child doesn't need to know the whole application.

## Start with a reusable form

This small form owns editable data, not whether it is adding or editing a record:

```swift
import ComposableArchitecture

@Reducer
struct EditForm {
    @ObservableState
    struct State: Equatable {
        var title = ""
    }

    enum Action: BindableAction {
        case binding(BindingAction<State>)
    }

    var body: some ReducerOf<Self> {
        // Applies binding actions to state.
        BindingReducer()
    }
}
```

In a modern SwiftUI view, a bindable store exposes bindings such as `$store.title`. They still send binding actions through the reducer; this isn't an arbitrary mutation outside the architecture.

When custom logic also responds to bindings, compose `BindingReducer()` before the custom `Reduce` so that the latter sees the updated value. See [Clamping runs after the write](#clamping-runs-after-the-write) for what that ordering can and cannot do.

```swift
import SwiftUI

struct EditFormView: View {
    // @Bindable derives bindings from the store supplied by the parent.
    @Bindable var store: StoreOf<EditForm>

    var body: some View {
        Form {
            TextField("Title", text: $store.title)
        }
    }
}
```

Use `let store` when only reading state and sending actions. Use `@Bindable` when an injected store needs binding projections. Neither creates a replacement store during view evaluation.

## Compose an always-present child

Using `EditForm` above:

```swift
@Reducer
struct FormScreen {
    @ObservableState
    struct State: Equatable {
        var form = EditForm.State()
        var savedTitle: String?
    }

    enum Action {
        case form(EditForm.Action)

        case saveButtonTapped
    }

    var body: some ReducerOf<Self> {
        Scope(state: \.form, action: \.form) {
            EditForm()
        }
        Reduce { state, action in
            switch action {
            case .form:
                return .none

            case .saveButtonTapped:
                state.savedTitle = state.form.title
                return .none
            }
        }
    }
}
```

`Scope` routes the child's portion of state and actions into its reducer. A scoped **store** gives a child view its runtime interface; reducer composition is separately necessary to run child logic.

Save belongs to this parent because it knows what committing means. Another presenter can reuse the same form to create a record, edit one, or display a draft without saving it.

The parent view can pass `store.scope(\.form, action: \.form)` to `EditFormView`. That view-level scoping doesn't replace the reducer's `Scope`.

If several sibling reducers must share a modifier such as `.ifLet` or `.forEach`, group them in `CombineReducers` and apply the modifier to the group. Without such a modifier, a reducer builder already combines siblings; an extra wrapper isn't needed.

## Bindings can trigger additional logic

This example uses generic binding actions for the slider controls and an explicit event for title edits:

```swift
@Reducer
struct BindingSettings {
    @ObservableState
    struct State: Equatable {
        var maximum = 10
        var title = ""
        var value = 0.0
    }

    enum Action: BindableAction {
        case binding(BindingAction<State>)

        case titleChanged(String)
    }

    var body: some ReducerOf<Self> {
        BindingReducer()
        Reduce { state, action in
            switch action {
            case .binding(\.maximum), .binding(\.value):
                // BindingReducer has already applied the edited value.
                state.maximum = min(20, max(1, state.maximum))
                state.value = min(Double(state.maximum), max(0, state.value))
                return .none

            case .binding:
                return .none

            case let .titleChanged(title):
                state.title = title
                return .none
            }
        }
    }
}

struct BindingSettingsView: View {
    @Bindable var store: StoreOf<BindingSettings>

    var body: some View {
        Form {
            // Sends .titleChanged rather than a generic binding action.
            TextField("Title", text: $store.title.sending(\.titleChanged))

            Stepper("Maximum: \(store.maximum)", value: $store.maximum, in: 1...20)

            Slider(value: $store.value, in: 0...Double(store.maximum))
                .accessibilityLabel("Value")
        }
    }
}
```

Lowering the maximum clamps the slider value in reducer logic, where it can be tested. The reducer also enforces the valid ranges if an action comes from somewhere other than these controls.

Binding `.sending(\.titleChanged)` chooses the action sent when a SwiftUI binding changes. **It is not Swift concurrency's `sending` keyword** and doesn't transfer an isolation region.

### `BindingReducer` has no per-field opt-out

`BindingReducer()` applies every `.binding` action it receives; it has no include or exclude list. The title field isn't special — it simply never sends a binding action. Bound with `$store.title`, it would send `.binding(\.title)`, `BindingReducer` would write the value, and `.titleChanged` would be dead code. The explicit action earns its place by giving the edit a home for logic and a name in logs and tests, not by making the write work. Matching `.binding(\.title)` in `Reduce` is enough if all you need is a hook after the write.

A field that is bound but never handled by `BindingReducer` is a bug: debug builds report a binding action that was not handled.

To exempt a group of fields, move them into a child whose action conforms to `BindableAction` and scope `BindingReducer` to that child:

```swift
var body: some ReducerOf<Self> {
    // Only `fields` accepts binding actions.
    Scope(state: \.fields, action: \.fields) {
        BindingReducer()
    }
}
```

The parent's action doesn't conform to `BindableAction`, so `$store.title` doesn't exist at that level and `.binding(\.title)` can't be constructed. Parent views send named actions instead.

### Clamping runs after the write

Reducers in `body` run in order, so with `BindingReducer()` first the custom `Reduce` sees the value already written. Clamping there is post-processing: the invalid value enters state and is corrected in the same reduce call, before any view renders. That is fine when the correction depends only on the new value and is idempotent, as these range checks are.

It cannot reject a write. Reversing the order lets `Reduce` see the old value, but `BindingReducer` still writes afterward, so a clamp placed there is overwritten.

To validate before the value lands, own the write:

```swift
switch action {
case let .maximumChanged(maximum):
    state.maximum = min(20, max(1, maximum))
    state.value = min(Double(state.maximum), max(0, state.value))
    return .none

case .binding:
    return .none
}
```

```swift
Stepper("Maximum: \(store.maximum)", value: $store.maximum.sending(\.maximumChanged), in: 1...20)
```

Prefer that when the rule needs the previous value, when the edit must be rejected rather than corrected, or for text input, where clamping after the write can fight the caret and input method. Sliders and steppers tolerate the snap back.

## Collections need stable identity

Use stable IDs when an operation refers to a row, especially across suspension. An array offset captured before `await` might point to a different row afterward. `IdentifiedArrayOf` preserves order while offering ID-based access.

For example, if `Document.ID` and `Folder.ID` are both aliases for `UUID`, those names don't create distinct ID types. Strongly typed IDs are a separate modeling choice.

See [Observing Collections](observing-collections.md) for an identified child-feature reducer and standard SwiftUI `ForEach`.

## Interaction state is still state

A form with a document title and editable checklist rows can own a focus enum with a title case and a row-ID case. Adding a row changes both data and focus, so an exhaustive test should assert both.

For a form that requires at least one row:

1. Normalize an empty draft by inserting a blank row.
2. When deletion empties the list, insert a replacement before selecting focus.
3. After deleting a middle or last row, focus a valid surviving row.
4. Generate IDs through a controlled UUID dependency, including IDs created in initializers.

SwiftUI focus needs a `FocusState` binding adapter. Keep the focus policy in reducer state and synchronize the view's focus property with it; don't let them become independent sources of truth.

**What I want to remember:** Reuse the child's behavior; let the parent own presentation and commit policy.

[Next: Presentation and Navigation](presentation-and-navigation.md) · [Back to TCA](index.md)
