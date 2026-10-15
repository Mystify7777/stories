# Chapter 5: Remembering What Was Finished

A task list is a view of the present. A productivity application also wants to tell a story about the past: what was completed today, how much estimated work was finished, and whether a week of effort is visible. The past becomes difficult when tasks can be completed, undone, deleted, cached, and changed on multiple devices.

HaloTask Pro's history module models one day's completed work as a set of tasks keyed by task ID. The count and total estimated minutes are derived from that set rather than stored as independent numbers. If a task appears twice during reconciliation, history should not count it twice simply because two devices reported it.

The module describes the server as the durable cross-device source of truth, with one row per user per day. IndexedDB is an offline cache and an outbox for changes the server has not acknowledged. Local keys are scoped to the authenticated user. Dates represent the user's local calendar day, supplied explicitly by the client rather than inferred from the server's clock. That matters near midnight and across time zones: a user's “today” should not change because a request reached a server in another region.

## Reconciliation instead of replacement

The merge operation combines task sets by ID. A local record can carry IDs this device explicitly removed because the user unchecked or deleted them. During reconciliation, the merge subtracts those known removals from the server set, then unions the remaining server tasks with local tasks. Metadata from a local duplicate wins, while the resulting count and minutes are recomputed from the merged set.

This distinguishes “I removed this task” from “I never saw this task.” If another device completed a task, the current device may not know about it yet. Its absence from the current local list is not evidence that it should be deleted from shared history. The implementation tracks IDs derived from its own task list to avoid treating unfamiliar remote work as a local removal.

When there is no pending local change, the server record wins. When a local snapshot is pending, the client merges rather than replacing the server row wholesale. If the server has no record but local history contains real data, the client can preserve and push it. The code comments state that every push is preceded by a pull and merge, and that pending state clears only when the server acknowledges the exact revision pushed. If another edit occurs while a push is in flight, the newer revision remains pending.

This is a small synchronization protocol, not just an analytics table. It has a merge function, revision tracking, a removal set, an acknowledgement rule, and a seven-day retention window.

## The cap and the evidence boundary

The implementation caps completed tasks at 500 per day and mirrors validation bounds between client and server. A cap protects the API and storage boundary, but it also means the product needs a deliberate policy for unusually large histories. The snapshot builder takes the first 500 tasks it receives; without inspecting all callers and their ordering, this book does not claim that selection is a user-visible prioritization guarantee.

The repository contains client history tests and server-side history route and validator tests. Their presence indicates that history is treated as a correctness surface rather than a decorative dashboard feature. The exact test guarantees still need to be read from assertions and, for the full server path, validated in the relevant environment. This chapter's implementation description comes from source contracts and functions, not from an independently executed test run.

### Evidence trail

- [History model and reconciliation](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-client/src/offline/history.ts)
- [History reconciliation tests](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-client/src/offline/history.test.ts)
- [History controller](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/src/controllers/history.controller.ts)
- [Server history validators](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/src/utils/historyValidators.ts)

### The broader lesson

When the same user can act from multiple devices, a summary number is not the source data. The source data is the set of facts from which the number can be recomputed, along with enough information to distinguish a deliberate removal from a missing observation.
