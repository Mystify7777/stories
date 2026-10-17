# Chapter 8: Importing a Session Means Accepting Untrusted History

Export and import look like two halves of a simple JSON feature. They are not symmetric in risk. Export serializes events that already exist in the Store. Import accepts data from outside the running process, where values may be malformed, forged, duplicated, oversized, or incompatible with the current event schema.

ADR-0011 treats import as a trust-boundary change, not a file-picker task.

## Preserve identity or do not call it restoration

Each event has an ID and timestamp. Those values describe historical identity and order. Routing imported data through `EventBus.report()` would regenerate missing identity/time values and could change the original record. Import therefore bypasses the Bus and inserts validated events directly into the Store.

The intended round-trip invariant is:

```text
A → export(A) → import(export(A)) → B
A[i] ≡ B[i] for every i
```

The original IDs, timestamps, and serialized order must survive. Import is not a re-capture operation; it restores a saved diagnostic session.

## Validate everything before mutating anything

The import contract parses JSON, requires an array, validates the closed version-1 event shape, checks required field types and severity values, rejects unknown fields, checks IDs for duplicates within the batch and collisions with the Store, and deep-freezes accepted events. It also rejects imports into a non-empty Store and rejects batches larger than the Store's capacity.

The order of operations is essential. If validation occurs while inserting, a malformed event late in the file could leave earlier events committed, creating a partial session. The specification requires the complete input to be validated first, then inserted through `EventStore.addMany()`. A failure must leave the Store untouched.

## Why unknown fields are rejected

The import schema is deliberately closed for known event fields. Silently accepting an extra field could create ambiguous behavior between versions or allow arbitrary data to enter the event model. At the same time, the category value remains open-string for plugin extensibility. This is a distinction between allowing new categories inside a known structure and allowing arbitrary structure to bypass validation.

The API returns a discriminated result: success reports the number of imported events; failure returns a structured error code and a human-readable message, with event index and field where applicable. Ordinary invalid input should not require callers to catch an exception just to display a validation failure.

## Why the Store must be empty

Importing into a non-empty Store would require merge semantics: how to resolve ID collisions, how to order old and new events, and what to do when capacity is exceeded. The current contract avoids pretending those decisions are already solved. It requires an empty Store and rejects over-capacity batches before insertion.

That is a constrained first version, not a claim that merging is impossible. The ADR explicitly leaves a dedicated import package for reconsideration if the capability grows independent consumers or substantial complexity.

## The security and correctness trade-off

Strict validation adds code and makes some future schema changes more deliberate. In return, it protects the Store's invariants and gives import failures a predictable, non-partial outcome. Session import is one of the clearest examples in DevLens of architecture being driven by a concrete failure mode rather than abstraction for its own sake.

**Sources:** [Session Import ADR](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0011-session-import.md), [Session Import specification](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/specs/session-import.md), [import implementation](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/panel/src/import.ts), [EventStore implementation](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/core/src/store.ts).