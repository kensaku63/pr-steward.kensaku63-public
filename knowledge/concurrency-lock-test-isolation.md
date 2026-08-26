# Concurrency lock test isolation

## Principle

A concurrency test only proves the lock or uniqueness boundary it is meant to exercise when earlier serialization mechanisms do not already force the requests into sequence.

If a create path takes a scope-local row lock before a repository-wide advisory lock, sending both requests to the same scope can produce deterministic replay or conflict even when the repository-wide lock is removed or broken. The test is green, but it does not isolate the intended safety mechanism.

## Review method

1. Trace the production lock acquisition order before evaluating the test.
2. Identify every earlier lock, unique constraint, queue, or single-threaded executor that can serialize the requests.
3. Build concurrent fixtures that do not share those earlier serialization keys while still sharing the identity governed by the target lock.
4. Start requests with a barrier and assert the externally visible convergence contract.
5. Query the authoritative repository-wide state, not only one request scope, when the identity is globally unique.

For a global idempotency identity guarded after a scope-local row lock, a strong conflicting-payload test uses two active scopes, one shared client identity, different payloads, and separate request paths. It should assert one durable creation, one canonical conflict, and exactly-once side effects across both scopes.

## Evidence boundary

Passing a same-scope race test can still be useful for scope-local sequencing or replay behavior. Do not delete it merely because it does not prove the global lock. Keep it for the contract it actually covers, and add or reshape a separate test that isolates the wider lock boundary.

Instrumentation or production seams are unnecessary when fixture separation can remove the confounding lock. Prefer that smaller test-only proof when it matches the public contract.
