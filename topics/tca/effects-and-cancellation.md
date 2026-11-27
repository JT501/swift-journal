---
title: "TCA Effects and Cancellation"
---

A reducer updates state synchronously. Networking, clocks, speech recognition, and file access need interaction with the outside world. An **effect** performs that work and sends an action back so the reducer can decide how state changes.

The loop is:

```text
Action → Reducer → New state + Effect
                          Effect → Response action → Reducer
```

Don't capture the reducer's `inout state` in an escaping effect. Capture the values the operation needs, and send a response action instead of mutating feature state from the effect.

## A controllable timer

```swift
import ComposableArchitecture

@Reducer
struct TimerCounter {
    @ObservableState
    struct State: Equatable {
        var count = 0
        var isRunning = false
    }

    enum Action {
        case timerTicked

        case toggleTimerButtonTapped
    }

    private enum CancelID {
        case timer
    }

    @Dependency(\.continuousClock) var clock

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .timerTicked:
                state.count += 1
                return .none

            case .toggleTimerButtonTapped:
                state.isRunning.toggle()
                guard state.isRunning else {
                    // Flipping the flag alone wouldn't stop the running effect.
                    return .cancel(id: CancelID.timer)
                }

                return .run { [clock] send in
                    for await _ in clock.timer(interval: .seconds(1)) {
                        await send(.timerTicked)
                    }
                }
                .cancellable(id: CancelID.timer, cancelInFlight: true)
            }
        }
    }
}
```

The effect waits for time to pass; only the reducer increments the count. Using a clock dependency instead of `Task.sleep` lets a test advance virtual time. A typed cancellation ID avoids accidental string mismatches.

## Lifetime is part of correctness

- An explicit start/stop feature needs an explicit cancellation path.
- An effect started from a view's `.task` can be awaited with `store.send(.task).finish()` so its lifetime follows the view task.
- Presentation and stack composition can cancel child effects when their state is removed.
- Cancellation is cooperative. Underlying callback APIs must release their resources when their stream or task terminates.

When saving or loading fails, send a failure action and restore loading state. Silently ignoring a thrown error can leave the UI showing a spinner forever. Handle cancellation separately when it isn't a user-facing failure.

## Two long-lived streams must run concurrently

A feature that displays both a running timer and live status messages must listen to two long-lived streams. If it awaits the entire message stream before starting the timer, the timer may never start. Run independent streams concurrently, for example with a task group inside an effect, and send separate actions for ticks and messages.

Use controlled streams to test how the feature reacts to messages and errors; unit tests don't need a real server.

If an alert pauses a running task, decide whether to ignore tick actions or actually cancel the timer effect. Ignoring ticks stops state updates, not the timer itself. If the alert offers Save and Discard, send a committed result to the parent only for Save; test those as distinct flows.

**What I want to remember:** Effects do the work; actions bring the results back; reducers own the mutations.

[Next: Composition and Bindings](composition-and-bindings.md) · [Back to TCA](index.md)
