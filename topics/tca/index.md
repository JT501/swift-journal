---
title: "TCA: The Composable Architecture"
---

The Composable Architecture (TCA) organizes a feature around state, actions, a reducer, and a store. These notes start from small counters, forms, and lists. No video or sample app is required.

## Reading order

1. [Foundations](foundations.md) — state, actions, reducers, and the store.
2. [Effects and Cancellation](effects-and-cancellation.md) — asynchronous work that feeds actions back into the feature.
3. [Composition and Bindings](composition-and-bindings.md) — reuse child features without duplicating their logic.
4. [Presentation and Navigation](presentation-and-navigation.md) — optional child state, editing drafts, and dismissal.
5. [Stack Navigation](stack-navigation.md) — model a navigation path and coordinate sibling features.
6. [Domain Modeling](domain-modeling.md) — use enums and stable IDs to rule out invalid states.
7. [Dependencies](dependencies.md) — control networking, permissions, time, and randomness.
8. [Testing](testing.md) — assert state changes, effect responses, and feature integration.
9. [Persistence](persistence.md) — save deliberately without leaking real file-system state into previews or tests.
10. [Observation and Identity](observation-and-identity.md) — property-read dependencies, conditional observation, and child-state replacement.
11. [Observing Collections](observing-collections.md) — distinguish collection structure from individual row contents.

## Version context

**The code here targets TCA 1.26.2.** Older TCA 1.0 examples use different spelling for some APIs, so don't paste these snippets unchanged into a project pinned to 1.0.0.

| Older examples may use | These notes use |
|---|---|
| Explicit `Reducer` conformance and hand-written boilerplate | `@Reducer` |
| `ViewStore` / `WithViewStore` | Observable state and direct store observation |
| `@BindingState` | Ordinary state properties with `@ObservableState` |
| `@PresentationState` | `@Presents` |
| Slash-style action case paths | Macro-generated case key paths |
| XCTest test methods | Swift Testing functions with the same `TestStore` concepts |

Where an example uses a feature defined in another note, it links to that definition.

The examples target TCA 1.26.2 in Swift 6 language mode with nonisolated default isolation. SwiftUI examples use native observation on iOS 17/macOS 14 or later; see the observation note for backport caveats.

The SwiftUI snippets compile in the validation package, and all eight reducer/observation tests pass. This doesn't verify rendered layouts or UI interaction.

[Back to the home page](../../index.md)
