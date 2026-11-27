---
title: "TCA Persistence: Loading, Saving, and Durability"
---

Persistence is the app's policy for which data survives a relaunch and how it is loaded, updated, and saved. The file system is just one service used to implement that policy. Keep persistence decisions in reducer behavior and file I/O behind a dependency; don't read or write shared simulator files in test setup.

## Save after the relevant mutation

A “root-level” hook lives in the app's top-level reducer, which owns or can access the data to save. For example, if child screens add and edit notes, the app reducer can save the updated notes list after those child actions run. The child screens don't each need their own file-writing code.

This is one design, not a requirement to save after every action. Save only when persisted data changes. Usually, save durable app data such as notes, but not temporary interface state such as whether an edit sheet or alert is currently open, or a timer's current tick. On relaunch, those temporary values can start fresh. Persist them too if the app specifically needs to restore them.

When starting an asynchronous save, pass it a snapshot of the data at that moment (for example, a copy of the notes array). Don't have the effect later reach back into the store for changing state, or capture the reducer's temporary `inout` state.

## Example: save when the notes list changes

Suppose the app's root state owns the list of notes. Inject an async save operation as a dependency, then attach `onChange` after the reducer logic (or after a `CombineReducers` containing child features). In TCA 1.26, the closure receives the new whole state:

```swift
@Dependency(\.notesPersistence.save) var saveNotes

var body: some ReducerOf<Self> {
    Reduce { state, action in
        switch action {
        case let .noteAdded(note):
            state.notes.append(note)
            return .none

        case .saveFailed:
            state.saveError = "Couldn't save notes."
            return .none
        }
    }
    .onChange(of: \.notes) { _, newState in
        // Capture the changed data now, not the mutable reducer state.
        .run { [notes = newState.notes, saveNotes] send in
            do {
                try await saveNotes(notes)
            } catch {
                await send(.saveFailed)
            }
        }
    }
}
```

Here, `notesPersistence.save` is an async dependency that writes the supplied array. The hook runs only when `notes` changes; opening an editor or showing an alert won't save anything. If child reducers can change `notes`, put them in the reducer composition wrapped by this `onChange`, so the root observes their updates too.

## Debounce a burst of changes

Using a storage interface, an effect helper can cancel a pending save and replace it with the newest snapshot:

```swift
import ComposableArchitecture
import Foundation

struct DataClient: Sendable {
    var load: @Sendable () throws -> Data
    var save: @Sendable (Data) throws -> Void
}

enum SaveResult {
    case saveFailed
}

private enum SaveCancelID {
    case debounce
}

func debouncedSave(
    _ notes: [String],
    storage: DataClient,
    clock: any Clock<Duration>
) -> Effect<SaveResult> {
    .run { send in
        do {
            // `notes` is the snapshot supplied when the reducer made this effect.
            try await clock.sleep(for: .seconds(1))
            try storage.save(JSONEncoder().encode(notes))
        } catch is CancellationError {
            // A newer snapshot replaced this pending save.
        } catch {
            await send(.saveFailed)
        }
    }
    .cancellable(id: SaveCancelID.debounce, cancelInFlight: true)
}
```

This is an effect helper, not a complete persistence feature. A reducer returns it for relevant changes and handles `.saveFailed`, for example by showing a retry option. In an app, register the storage client as a dependency and supply a controlled clock.

Debouncing means waiting for a quiet period, not guaranteeing a save every second. Continuous edits can postpone saving indefinitely. Decide how pending changes are flushed at lifecycle boundaries; cancelling a pending save doesn't make those changes durable.

## Load, save, and recover

Persistence is a lifecycle: load saved data when the feature starts, update it in response to user actions, then save the updated durable data. For small synchronous storage, loading in an initial-state factory is one option. For asynchronous or larger storage, send a load action and perform the read in an effect so the UI can represent loading and failure.

Treat corrupt or unreadable data as an explicit recovery decision. For example, show an error and offer to reset, or intentionally fall back to an empty list. Don't silently report a failed load as success unless that is the product's recovery policy.

Likewise, decide what a save failure means: show a retry, keep a dirty/unsaved indicator, or retry according to a deliberate policy. If saves are debounced, decide how pending changes are flushed when the app backgrounds or exits. Cancelling a pending save does not make it durable.

Use isolated in-memory storage for previews and tests rather than reading or writing the developer's real files; see [Dependencies](dependencies.md#choose-implementations-by-context).

Foundation's synchronous file APIs block their calling thread. Real storage should use an appropriate I/O execution strategy; wrapping blocking disk work in an async closure doesn't automatically move it off the main actor.

Reducer tests verify the app's persistence policy. Integration tests should also verify real encoding, file paths, and the live storage implementation. UI tests need clean storage too—weakening an expected count from “one” to “at least one” can hide a duplicate-insertion bug.

**What I want to remember:** Own the persistence boundary, isolate test storage, and distinguish a pending save from a durable one.

[Back to TCA](index.md)
