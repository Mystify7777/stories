# Chapter 5: Remembering What Was Finished

A task list is a view of the present. A productivity application also wants to tell a story about the past: what was completed today, how much estimated work was finished, and whether a week of effort is visible. The past becomes difficult when tasks can be completed, undone, deleted, cached, and changed on multiple devices.

HaloTask Pro's history module models one day's completed work as a set of tasks keyed by task ID. Count and total estimated minutes are recomputed from that set rather than stored as independent numbers. If a task appears twice during reconciliation, history should not count it twice simply because two devices reported it.

The module describes the server as the durable cross-device source of truth, with one row per user per day. IndexedDB is an offline cache and an outbox for changes the server has not acknowledged. Local records are scoped to the authenticated user. Calendar dates are the user's local dates, declared by the client along with `utcOffsetMinutes`. But the server does not simply trust that declaration: it uses its own clock shifted by the supplied offset to validate what “today” means.

That small detail becomes important around midnight. The API accepts exactly one “today” for a declared offset, with no clock-skew tolerance. A payload arriving a moment after the browser's local day has changed may be rejected as `DATE_NOT_TODAY`, even when its snapshot was valid when created. A historical snapshot may also fall outside the accepted backfill window and be rejected with `DATE_OUT_OF_RANGE`. Those machine-readable date errors are different from malformed payload errors: the client treats the clock-dependent cases as retryable, while ordinary validation errors are not retried unchanged. The source therefore describes a date contract shared between a local calendar and a server clock, not a date chosen independently by either side.

## Reconciliation happens on the client

The merge operation combines task sets by ID. A local record can carry IDs that this device explicitly removed because the user unchecked or deleted them. During reconciliation, the merge subtracts those known removals from the server set, then unions the remaining server tasks with local tasks. Metadata from a local duplicate wins, and count and minutes are recomputed from the merged set.

This distinguishes “I removed this task” from “I never saw this task.” If another device completed a task, the current device may not know about it yet. Its absence from the current local list is not evidence that it should be deleted from shared history. The implementation tracks IDs derived from its own task list to avoid treating unfamiliar remote work as a local removal.

When there is no pending local change, the server record wins. When a local snapshot is pending, the client pulls the server state, merges it locally, then pushes a merged snapshot if necessary. The source comments say that every push is preceded by a pull and merge, and that pending state clears only when the server acknowledges the exact revision pushed. If another edit occurs while a push is in flight, the newer revision remains pending.

The location of that merge is critical. The server's history endpoint validates the snapshot, then uses an upsert filtered by authenticated user and date with a `$set` of the snapshot fields. The write replaces the stored per-day snapshot; it does not inspect the stored task set and merge records from two devices. The client performs the merge before it calls the endpoint. That is a coherent client-led reconciliation design, but it is not server-side conflict resolution, and the upsert alone does not protect against two devices that both pull the same old snapshot and then push different merged results. The repository's own engineering log names this remaining race explicitly.

## A deliberate boundary at the clock

The client sends the date and offset for each history request. The server uses `getToday(offset, serverNow)` for the “today” endpoint and GET window, and validates backfill dates against the same offset-adjusted server date and a seven-day range. A date mismatch caused by the moving clock can be retried by the client later. A malformed snapshot, duplicate task ID, inconsistent count, or incorrect total minutes is a payload problem and is rejected rather than treated as temporary clock skew.

This distinction matters because queued history can outlive the calendar day in which it was recorded. The history module chooses `/today` only when the entry date still matches the client's current local date; otherwise it uses the bounded `/:date` backfill route. The server still validates the request, so the route choice is not permission to write arbitrary historical dates.

## The cap and the evidence boundary

The implementation caps completed tasks at 500 per day and mirrors validation bounds between client and server. A cap protects the API and storage boundary, but the product still needs a policy for unusually large histories. The client snapshot builder takes the first 500 eligible tasks it receives; without a verified ordering guarantee from its callers, this book does not claim which tasks survive the cap is a user-visible prioritization rule.

The repository contains client history tests and server-side history route and validator tests. The server route tests use a mocked model with the real router, middleware, controller, and validators. They establish validation, user scoping, status codes, and the shape of database calls under the test double. They do not establish durable convergence for concurrent writes against a real MongoDB deployment. The client tests cover simulated merge and retry behavior; that is useful convergence coverage, but not proof that every real interleaving between two devices is safe.

### Evidence trail

- [Client history implementation, pinned to the reviewed source revision](https://github.com/Mystify7777/Halotask-Pro/tree/e95ba08b107dcb0bd7e2941850bfb8246727b309/halotasks-client/src/offline/history.ts)
- [History controller, pinned to the reviewed source revision](https://github.com/Mystify7777/Halotask-Pro/tree/e95ba08b107dcb0bd7e2941850bfb8246727b309/halotasks-server/src/controllers/history.controller.ts)
- [Calendar contract, pinned to the reviewed source revision](https://github.com/Mystify7777/Halotask-Pro/tree/e95ba08b107dcb0bd7e2941850bfb8246727b309/halotasks-server/src/utils/calendarDate.ts)
- [History validators, pinned to the reviewed source revision](https://github.com/Mystify7777/Halotask-Pro/tree/e95ba08b107dcb0bd7e2941850bfb8246727b309/halotasks-server/src/utils/historyValidators.ts)
- [Client history tests, pinned to the reviewed source revision](https://github.com/Mystify7777/Halotask-Pro/tree/e95ba08b107dcb0bd7e2941850bfb8246727b309/halotasks-client/src/offline/history.test.ts)
- [Server history route tests, pinned to the reviewed source revision](https://github.com/Mystify7777/Halotask-Pro/tree/e95ba08b107dcb0bd7e2941850bfb8246727b309/halotasks-server/tests/historyRoutes.test.ts)
- [History synchronization development notes, pinned to the reviewed source revision](https://github.com/Mystify7777/Halotask-Pro/tree/e95ba08b107dcb0bd7e2941850bfb8246727b309/docs/logs.md)

### The broader lesson

When the same user can act from multiple devices, a summary number is not the source data. The source data is the set of facts from which the number can be recomputed, plus enough information to distinguish a deliberate removal from a missing observation. Client-side reconciliation makes offline recovery possible, but unless the server participates in conflict resolution, two clients can still race between a pull and a replacing write.
