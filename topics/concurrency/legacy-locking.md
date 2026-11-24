---
title: Legacy Locking
---

Imagine a bank that stores accounts in a shared dictionary. Each account has a mutable balance. The bank needs to support opening accounts, transferring money between them, and reading the total across all accounts. If multiple threads can call these operations, the bank needs a way to protect its shared state.

## What is `NSLock`?

`NSLock` allows only one thread at a time to hold a given lock. The code that runs while holding it is called a *critical section*. If every operation that reads or writes the accounts dictionary uses the same lock, one thread can't change the dictionary while another is reading it. `withLock` releases the lock when its closure returns or throws.

The lock protects the operation, not anything it returns. If the bank hands out a mutable account object, the caller can still change its balance after the lock has been released. Returning a balance as a value avoids exposing that mutable reference.

## Why isn't adding `NSLock` enough?

Putting a lock around each method can stop the original dictionary race, but it doesn't make the bank safe by itself. Every access has to follow the same locking rule, and Swift doesn't enforce that rule for `NSLock`. Methods that work safely on their own may also deadlock when one calls the other while holding the lock.

If you use `NSLock`, give the whole operation one clear lock boundary and keep mutable references from escaping it. For the bank, that means protecting the lookup, debit, and credit together, and returning values such as balances rather than mutable account objects.

## What is a deadlock?

A deadlock happens when work can't proceed because the operations involved are waiting on one another. A single thread can deadlock itself: suppose a transfer takes the bank's lock so it can look up both accounts, debit one, and credit the other as one operation. If it calls a lookup method that tries to take the same `NSLock`, the lookup waits for the transfer to release it, and the transfer waits for the lookup to return.

## What is `NSRecursiveLock`?

`NSRecursiveLock` lets the same thread acquire the same lock more than once. Every acquisition needs a matching unlock, and other threads remain blocked until the final unlock. This avoids that particular self-deadlock, not deadlocks in general.

## Why can a recursive lock be a code smell?

Consider a transfer of 50 dollars between two accounts that each start with 100 dollars. The bank's total should be 200 dollars before and after the transfer. Now suppose debiting the first account immediately calls a notification callback, and that callback asks the bank for its total. With a recursive lock, the callback can acquire the lock on the same thread and read **150 dollars**: the debit has happened, but the credit hasn't.

No other thread got through the lock. The problem is that the callback was allowed to observe state in the middle of an update.

**What I want to remember:** Keep the lock boundary clear. When I control the code, I'd rather acquire the lock once in the outer operation and notify callers after the update is complete than use a recursive lock to make nested calls work.

[Back to Swift Concurrency](index.md)
