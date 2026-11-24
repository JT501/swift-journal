---
title: Swift Concurrency Foundations
---

Four questions I used to get a better handle on Swift concurrency.

## What is isolation?

Isolation is about which code can access mutable state safely. Swift checks isolation boundaries at compile time. The point isn't to pin the work to one thread; it's to prevent unsafe concurrent access to the state.

## What is an actor?

An actor is a reference type that protects its mutable state. Only one piece of actor-isolated code runs on a given actor at a time. If a method suspends at an `await`, other work can run on the actor before that method resumes.

## What is a data race?

A data race happens when two operations access the same memory concurrently without synchronization and at least one of them writes. For example, opening bank accounts one at a time works, but opening them from many tasks at once can cause simultaneous writes to the same dictionary. A passing sequential test doesn't show that the code is safe to use concurrently.

## What is a race condition?

A race condition is a bug whose outcome depends on the timing or order of operations. Imagine an actor method checks a balance, suspends while waiting for approval, then debits the account. Another operation can change the balance while the first is suspended. The accesses are serialized, so there's no data race, but the original balance check may no longer be valid.

**What I want to remember:** Actor isolation can prevent data races, but it doesn't make a check and an update atomic across a suspension.

[Back to Swift Concurrency](index.md)
