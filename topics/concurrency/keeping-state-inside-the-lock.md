---
title: Keeping State Inside the Lock
---

Imagine a bank that stores mutable `Account` objects in a dictionary. An account has a balance that can change when someone deposits or withdraws money.

## Why isn't a locked lookup enough?

An `account(for:) -> Account` method can hold a lock while it looks up the account. But once the method returns, the caller has a reference to the same mutable account the bank uses. Reading its balance can race with a deposit on another thread, even if the deposit takes the lock.

A `balance(for:) -> Int` method reads the balance **while holding the lock** and returns a copy of the number. The number may go stale, but it doesn't give callers access to the account itself.

## What does `OSAllocatedUnfairLock` change?

`OSAllocatedUnfairLock` can store the state it protects. Instead of keeping a lock and a dictionary in separate properties, put the dictionary inside the lock and access it through `withLock`.

The checked `withLock` API requires a `Sendable` result. `Int` qualifies; a plain mutable `Account` class doesn't. That gives the compiler a way to catch this particular mistake instead of relying on a convention that callers must remember.

## Why doesn't `@unchecked Sendable` fix the race?

Adding `@unchecked Sendable` tells the compiler to trust that you've made the type safe to share. It doesn't add a lock or change how the program runs. If callers can still reach the same mutable balance without synchronization, the race is still there.

**What I want to remember:** Don't just lock the lookup. Read the value I need inside the lock, and return the value rather than the mutable object.

[Back to Swift Concurrency](index.md)
