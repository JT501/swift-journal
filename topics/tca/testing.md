---
title: "Testing TCA Features"
---

A `TestStore` runs a reducer and observes both state transitions and effect actions. In its default exhaustive mode, tests must account for all state changes, received actions, and effects still running at the end.

The examples use Swift Testing with TCA's `TestStore`. Its send/receive model works with either Swift Testing or XCTest.

## Assert a synchronous transition

Using [Counter](foundations.md):

```swift
import ComposableArchitecture
import Testing

@MainActor
@Test
func incrementCounter() async {
    let store = TestStore(initialState: Counter.State()) {
        Counter()
    }

    await store.send(.incrementButtonTapped) {
        // Describe the expected new state, not a mutation of the real store.
        $0.count = 1
    }
}
```

The assertion closure starts from the previous state and describes the expected next state. If some other field changes unexpectedly, an exhaustive test fails too.

## Control the request and receive its response

Using [FactLoader and FactClient](dependencies.md):

```swift
@MainActor
@Test
func loadFact() async {
    let store = TestStore(initialState: FactLoader.State()) {
        FactLoader()
    } withDependencies: {
        $0.factClient.fetch = { "\($0) is the answer." }
    }

    await store.send(.fetchButtonTapped) {
        $0.isLoading = true
    }
    await store.receive(\.factReceived) {
        $0.fact = "42 is the answer."
        $0.isLoading = false
    }
    await store.finish()
}
```

`send` represents input into the feature; `receive` asserts an action produced by an effect. A deterministic service response makes the test about the feature rather than the network.

Also test a throwing client: it should produce `.factFailed` and reset `isLoading`. Not handling the error should fail a test, not be hidden by weakening the assertion.

## Advance virtual time, not real time

Using [TimerCounter](effects-and-cancellation.md):

```swift
@MainActor
@Test
func startAndStopTimer() async {
    let clock = TestClock()
    let store = TestStore(initialState: TimerCounter.State()) {
        TimerCounter()
    } withDependencies: {
        $0.continuousClock = clock
    }

    await store.send(.toggleTimerButtonTapped) {
        $0.isRunning = true
    }
    await clock.advance(by: .seconds(1))
    await store.receive(\.timerTicked) {
        $0.count = 1
    }
    await store.send(.toggleTimerButtonTapped) {
        $0.isRunning = false
    }
    await store.finish()
}
```

An infinite timer must be cancelled before the test ends. Don't replace virtual time with a real sleep plus a safety margin: that makes tests slower and timing-dependent.

## Test parent-child integration with `TestStore`

This is a reducer test, not a UI test: `TestStore` sends actions directly, without rendering views or tapping controls. Initialize the parent in its starting state, send the actions for opening and editing a draft, then verify that saving commits it and closes the sheet. For features that create IDs, use fixed UUID and date dependencies so tests are repeatable.

Non-exhaustive tests can focus on a final business outcome by opting out of per-action state and effect assertions. The actions still run, and explicit assertions on `store.state` still work, but unexpected intermediate changes or effect actions can go unnoticed. Keep exhaustive tests where practical; don't disable exhaustivity just to hide behavior you haven't understood.

Using [DocumentEditor and its form](presentation-and-navigation.md):

```swift
@MainActor
@Test
func saveEditedTitle() async {
    let store = TestStore(initialState: DocumentEditor.State()) {
        DocumentEditor()
    }
    await store.send(.editButtonTapped) {
        $0.edit = EditForm.State(title: "Original")
    }
    await store.send(
        .edit(.presented(.binding(.set(\.title, "Revised"))))
    ) {
        $0.edit?.title = "Revised"
    }
    await store.send(.saveButtonTapped) {
        $0.title = "Revised"
        $0.edit = nil
    }

    // The intermediate assertions also verify draft creation and editing.
    #expect(store.state.title == "Revised")
    #expect(store.state.edit == nil)
}
```

This exhaustive reducer test covers parent/child integration by sending actions directly. A non-exhaustive version could omit the assertion closures and set `store.exhaustivity = .off` to focus only on the final outcome, but would no longer verify each draft transition.

The test above only covers Save. Cancel is a different path: it should close the draft without copying its edited title into the parent. Test that separately:

```swift
await store.send(.editButtonTapped) {
    $0.edit = EditForm.State(title: "Original")
}
await store.send(
    .edit(.presented(.binding(.set(\.title, "Unsaved change"))))
) {
    $0.edit?.title = "Unsaved change"
}
await store.send(.cancelButtonTapped) {
    $0.edit = nil
}
#expect(store.state.title == "Original")
```

This checks that cancelling discards the draft rather than committing it.

## Test interactive presentation dismissal

This verifies the action TCA receives when SwiftUI dismisses `DocumentEditor`'s sheet interactively, for example, when the user swipes it down:

```swift
@MainActor
@Test
func dismissEditDraft() async {
    let store = TestStore(
        initialState: DocumentEditor.State(edit: EditForm.State(title: "Unsaved"))
    ) {
        DocumentEditor()
    }

    await store.send(.edit(.dismiss)) {
        $0.edit = nil
    }
    #expect(store.state.title == "Original")
}
```

The presentation reducer clears the draft; the saved title remains unchanged.

## Test binding policy

Using [BindingSettings](composition-and-bindings.md), verify the clamp rather than relying on a slider to look correct:

```swift
@MainActor
@Test
func loweringMaximumClampsValue() async {
    let store = TestStore(initialState: BindingSettings.State(value: 8)) {
        BindingSettings()
    }

    await store.send(.binding(.set(\.maximum, 5))) {
        $0.maximum = 5
        $0.value = 5
    }
}
```

This tests reducer behavior. It doesn't prove the actual slider is wired correctly or that the sheet's animation behaves correctly.

## Observation correctness is a different test

A `TestStore` can prove that state changed correctly without proving that a view subscribed to the right properties. Using [ObservationDemo](observation-and-identity.md), a small observation test can check dependency tracking directly:

```swift
import Observation

@MainActor
@Test
func hiddenSummaryDoesNotObserveCount() {
    let store = Store(initialState: ObservationDemo.State()) {
        ObservationDemo()
    }
    let changes = LockIsolated(0)

    withObservationTracking {
        if store.isShowingSummary {
            _ = store.child?.count
        }
    } onChange: {
        changes.withValue { $0 += 1 }
    }

    store.send(.child(.incrementButtonTapped))
    #expect(changes.value == 0)

    store.send(.toggleSummaryButtonTapped)
    #expect(changes.value == 1)
}

@MainActor
@Test
func replacingChildUpdatesObservedCount() {
    let store = Store(
        initialState: ObservationDemo.State(child: Counter.State(count: 3))
    ) {
        ObservationDemo()
    }
    let changes = LockIsolated(0)

    withObservationTracking {
        _ = store.child?.count
    } onChange: {
        changes.withValue { $0 += 1 }
    }

    // Replaces the child and changes the displayed count from three to zero.
    store.send(.resetChildButtonTapped)
    #expect(changes.value == 1)
    #expect(store.child?.count == 0)
}
```

`withObservationTracking` watches the observable properties read inside its first closure. When one changes, it runs `onChange` once; it doesn't keep calling that closure for later changes. To watch another change, call `withObservationTracking` again and read the properties again. The examples only check one change. These assertions test observation invalidation, not how many times SwiftUI evaluates a view body.

In this test, `.resetChildButtonTapped` resets the child to its initial state. The test starts at count `3`, so the reset changes the observed count to `0` and should notify the observer. If the child already had count `0`, resetting it to another state with count `0` changes no observable value; don't expect a notification just because the reducer assigned a newly constructed state.

This observation test checks which state reads can trigger invalidation; it doesn't check what SwiftUI renders or how often it evaluates a view. If a view updates unexpectedly, temporary logging or a UI test can help you see whether its `body` is recomputing often or a store is being recreated. Fewer log messages alone don't prove the app is faster. If a snapshot test changes, inspect the new screenshot before accepting it; blindly recording it as the new expected image could hide a UI bug.

Reducer tests don't replace UI tests for layout, accessibility, or actual framework integration. UI tests also need isolated persistence; otherwise data from an earlier run can create misleading successes or failures.

**What I want to remember:** A fast test controls time, services, and storage, then asserts the app's decisions—not the outside world.

[Next: Persistence](persistence.md) · [Back to TCA](index.md)
