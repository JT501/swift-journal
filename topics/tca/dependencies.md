---
title: "TCA Dependencies: Control the Outside World"
---

A dependency is something the feature doesn't fully control: a network service, the current date, UUID generation, time passing, permissions, or the file system.

Calling these APIs directly makes behavior depend on the environment. A permission request can suspend forever in a preview that cannot display the system prompt. An unpredictable UUID or network response can make an otherwise simple test impossible to assert.

## Put a small interface in front of the service

A service that fetches a short fact about a number can be represented by a struct holding one operation. `@DependencyClient` generates defaults that report unimplemented endpoints in tests:

```swift
import ComposableArchitecture
import Foundation

@DependencyClient
struct FactClient: Sendable {
    var fetch: @Sendable (Int) async throws -> String
}

extension FactClient: DependencyKey {
    static let liveValue = Self(
        fetch: { number in
            guard let url = URL(string: "https://numbersapi.com/\(number)") else {
                throw URLError(.badURL)
            }

            let (data, response) = try await URLSession.shared.data(from: url)
            guard let response = response as? HTTPURLResponse,
                  (200..<300).contains(response.statusCode)
            else {
                throw URLError(.badServerResponse)
            }

            return String(decoding: data, as: UTF8.self)
        }
    )

    static let previewValue = Self(
        fetch: { "\($0) is our preview number." }
    )

    static let testValue = Self()
}

extension DependencyValues {
    var factClient: FactClient {
        get { self[FactClient.self] }
        set { self[FactClient.self] = newValue }
    }
}
```

The live implementation depends on an external service and network availability; it isn't used to verify reducer behavior. Tests replace it with a deterministic closure. Unlike a single-response request, a speech recognizer can expose an `AsyncThrowingStream<String, Error>` so callers receive many updates.

You can also write a dependency client by hand. The macro above reduces boilerplate; it isn't required for struct-based dependencies.

## Use the dependency in a feature

Using `FactClient` above:

```swift
@Reducer
struct FactLoader {
    @ObservableState
    struct State: Equatable {
        var fact: String?
        var isLoading = false
        var number = 42
    }

    enum Action {
        case factFailed

        case factReceived(String)

        case fetchButtonTapped
    }

    @Dependency(\.factClient) var factClient

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .factFailed:
                state.isLoading = false
                return .none

            case let .factReceived(fact):
                state.fact = fact
                state.isLoading = false
                return .none

            case .fetchButtonTapped:
                guard !state.isLoading else { return .none }

                state.fact = nil
                state.isLoading = true
                return .run { [number = state.number, factClient] send in
                    do {
                        let fact = try await factClient.fetch(number)
                        await send(.factReceived(fact))
                    } catch is CancellationError {
                        // No user-facing failure for cancellation.
                    } catch {
                        await send(.factFailed)
                    }
                }
            }
        }
    }
}
```

The number is captured when the request starts, rather than attempting to capture mutable `inout` state. Failure clears the spinner; a production UI could also store an error message and offer retry. This small example has no explicit cancel button; if one is added, its action should also clear loading state.

## Choose implementations by context

- **Live:** Call the real service on a device or simulator.
- **Preview:** Return useful sample results without permission prompts or real I/O.
- **Test:** Override only the endpoints needed for the scenario; unexpected use should be visible.

The same approach applies to built-in dependencies such as `@Dependency(\.uuid)`, `@Dependency(\.date.now)`, and `@Dependency(\.continuousClock)`. Once a dependency is declared, use it instead of reaching for the global API.

## Isolate storage in tests and previews

Storage is a dependency too. The `DataClient` interface in [Persistence](persistence.md) represents load and save operations. Give each test or preview its own in-memory implementation instead of touching real files:

```swift
import ComposableArchitecture
import Foundation

func makeInMemoryStorage(initialData: Data = Data()) -> DataClient {
    // LockIsolated protects the fake storage shared by these Sendable closures.
    let data = LockIsolated(initialData)
    return DataClient(
        load: { data.value },
        save: { data.setValue($0) }
    )
}
```

Each call creates independent storage. A test can save, reload, and assert exact values without leaking data across runs. Use throwing implementations when testing load or save failures; don't silently treat corrupted data as a successful empty load unless that is the intended recovery policy.

## Preview without changing production wiring

Because `FactClient` declares a `previewValue`, TCA selects it automatically when the store runs in an Xcode preview. No explicit override is needed for this example:

```swift
import SwiftUI

#Preview {
    let store = Store(initialState: FactLoader.State()) {
        FactLoader()
    }

    VStack {
        Button("Fetch fact") {
            store.send(.fetchButtonTapped)
        }
        Text(store.fact ?? "No fact yet")
    }
}
```

Tapping Fetch in the preview sends an action; the reducer uses `FactClient.previewValue` instead of making a network request. The same store construction in the running app uses `liveValue`. Tests use `testValue` unless they override the dependency.

Use an explicit override only when you want a different response for a particular preview or child feature. For example, `.dependency(\.factClient, customFactClient)` on a reducer inside `Scope` changes that child's implementation without changing siblings.

For a one-off preview, override the reducer directly:

```swift
#Preview("Custom fact") {
    let store = Store(initialState: FactLoader.State()) {
        FactLoader()
            .dependency(
                \.factClient,
                FactClient(fetch: { number in "\(number) is the custom fact." })
            )
    }

    Button("Fetch fact") {
        store.send(.fetchButtonTapped)
    }
}
```

This override takes precedence over `previewValue` for this feature. The regular preview above still uses `previewValue`, and the running app still uses `liveValue`.

**What I want to remember:** Test how the app responds to the service, not whether the external service happens to work today.

[Next: Testing](testing.md) · [Back to TCA](index.md)
