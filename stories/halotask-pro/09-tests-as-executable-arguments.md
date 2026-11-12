# Chapter 9: Tests as Executable Arguments

A test is most valuable when it explains which failure the system is trying to prevent. HaloTask Pro's offline and session tests do not stop at checking that a function returned an expected object. They construct situations in which identity, ordering, or acknowledgement could change between asynchronous steps.

The queue model tests use a map-backed replacement for IndexedDB while retaining the real queue module and authentication store. They verify that actions are stored under account-specific keys, that an action remains associated with the account that created it even if another account signs in before the queued write runs, and that enqueue refuses to proceed without an authenticated owner. They also exercise coalescing behavior and removal of the legacy unscoped queue.

The queue processor tests model 401 responses and verify that the refused entry and unattempted suffix remain in order. They test the distinction between a credential failure and a permanently invalid client request. The application should not destroy a user's valid intent because a token has become invalid.

Credential tests go closer to the boundary. They use the real task service and API client request interceptor with a recording adapter, then record the method, URL, and authorization header for requests that reach the adapter. This lets a test assert which credential would have gone on the wire, or that no request was sent after the session changed. That is stronger than mocking the service and checking only whether it was called.

The server's session-invalidation suite describes a Mongo-independent model that mimics database snapshots and atomic conditional updates where races matter. It uses gates to force requests into specific interleavings, including password-reset races. Its comments note that a real MongoDB round trip lives in another integration suite requiring a downloaded mongod binary and therefore may not run in every environment. A deterministic model test can test a race, but it is not the same as executing the full integration test against MongoDB.

## The path the project took

The repository's recorded development history shows a sequence of increasingly explicit correctness boundaries. This is a chronology of selected documented source milestones, not a claim that each feature was the only work done in its period.

- **April 19, 2026: offline foundations.** The project changelog records cache-first loading, a write queue, reconnect replay, pending indicators, and a manual retry control. That established the first local-intent path.
- **May 20, 2026: completed-task history.** The changelog records completion timestamps and a completed-task history section. Later, the [May 21 history snapshot commit](https://github.com/Mystify7777/Halotask-Pro/commit/f3bfe405282127be97d68d710ac4b81eaa082250) added seven-day history snapshots and the weekly insights chart.
- **May 26, 2026: history consistency across views.** Commits [208f06f](https://github.com/Mystify7777/Halotask-Pro/commit/208f06ff9e90f34853a86cf2479744d4c885ba31) and [fb6f5ad](https://github.com/Mystify7777/Halotask-Pro/commit/fb6f5ad3a50b8448f86be67c2ec2dee8b8724001) record improvements to persisted completed history and aligning insight cards to it.
- **September 30 to October 1: explicit ownership and reconciliation.** The engineering log records user-scoped Growth Tree storage on September 30 and the history source-of-truth, date-semantics, and retry redesign on October 1. The history log is unusually candid about its limit: two devices that pull and push in the same short GET-to-PUT window can still overwrite one another without a server-side merge.
- **October 3, 2026: revocable sessions.** [The token-versioning commit](https://github.com/Mystify7777/Halotask-Pro/commit/1f9dbeead169b7859317b70ce241e7de8f1cb855) added tokenVersion to server accounts and JWT claims, rejected stale generations in middleware, and changed client queue handling to preserve queued writes after a 401 while binding them to the original account and token.
- **October 4, 2026: security and operational boundaries.** The log records production HTTP security headers, more explicit MongoDB startup validation and DNS fallback behavior, and graceful shutdown work. The descriptions also state their limits: unit and model-double coverage does not replace a real Atlas connection or a deployed header check.
- **October 5, 2026: trust-boundary validation.** [The validation and API test commit](https://github.com/Mystify7777/Halotask-Pro/commit/a6cce95a8b52f69e56aa9c0acb660429a4db0182) records runtime JWT-payload validation, request-body guards, stronger task ownership tests, and session invalidation tests.
- **October 8, 2026: server-authoritative Growth Tree.** [The award-system commit](https://github.com/Mystify7777/Halotask-Pro/commit/e95ba08b107dcb0bd7e2941850bfb8246727b309) records tests for task completion, XP awards, tree rules, concurrency, and error handling. The accompanying change log explains the atomic XP/ledger write and recovery marker used when derived streak persistence fails.
- **October 6, 2026: AI provider update.** The log also records migration to a newer Groq-hosted model, including explicit uncertainty that output-budget adequacy still needs live provider validation. This is a smaller but useful example of separating deterministic request-shape tests from external service behavior.

Read chronologically, these milestones shift the narrative. The initial goal was keeping task entry available when disconnected. Over time, the harder issue became deciding which user owns pending work, which system owns the durable truth, what happens when data crosses a date boundary, and how to recover if one half of a multi-step operation succeeds while another does not.

## What a test suite can and cannot prove

A test can demonstrate that a particular input and interleaving lead to an expected result under its assumptions. It does not prove the absence of all races, validate production environment variables, or establish that an external provider is available. A passing test count is a dated observation, not a permanent property of a repository.

The client and server manifests define verification scripts. The client runs Vitest and a production build. The server runs Vitest, source type-checking, test type-checking, and a build. The root script delegates to those package-level verification commands. This is a verification contract, not evidence that the commands passed during this writing session. They were not executed for this book.

A stronger verification record would include the exact commit SHA, command, environment, result, and tests skipped because services were unavailable. That lets readers distinguish code review from executed verification and local verification from integration or production checks.

### Evidence trail

- [Queue tests](https://github.com/Mystify7777/Halotask-Pro/tree/e95ba08b107dcb0bd7e2941850bfb8246727b309/halotasks-client/src/offline/syncQueue.test.ts)
- [Queue processor tests](https://github.com/Mystify7777/Halotask-Pro/tree/e95ba08b107dcb0bd7e2941850bfb8246727b309/halotasks-client/src/offline/queueProcessor.test.ts)
- [Credential boundary tests](https://github.com/Mystify7777/Halotask-Pro/tree/e95ba08b107dcb0bd7e2941850bfb8246727b309/halotasks-client/src/offline/queueProcessor.credentials.test.ts)
- [Session invalidation tests](https://github.com/Mystify7777/Halotask-Pro/tree/e95ba08b107dcb0bd7e2941850bfb8246727b309/halotasks-server/tests/sessionInvalidation.test.ts)
- [Tree award unit/concurrency tests](https://github.com/Mystify7777/Halotask-Pro/tree/e95ba08b107dcb0bd7e2941850bfb8246727b309/halotasks-server/tests/treeAward.test.ts)
- [Exact Mongoose-command tests](https://github.com/Mystify7777/Halotask-Pro/tree/e95ba08b107dcb0bd7e2941850bfb8246727b309/halotasks-server/tests/treeAward.mongoose.test.ts)
- [Client verification scripts](https://github.com/Mystify7777/Halotask-Pro/tree/e95ba08b107dcb0bd7e2941850bfb8246727b309/halotasks-client/package.json)
- [Server verification scripts](https://github.com/Mystify7777/Halotask-Pro/tree/e95ba08b107dcb0bd7e2941850bfb8246727b309/halotasks-server/package.json)
