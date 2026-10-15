# Chapter 9: Tests as Executable Arguments

A test is most valuable when it explains which failure the system is trying to prevent. HaloTask Pro's offline and session tests do not stop at checking that a function returned an expected object. They construct situations in which identity, ordering, or acknowledgement could change between asynchronous steps.

The queue model tests use a map-backed replacement for IndexedDB while retaining the real queue module and authentication store. They verify that actions are stored under account-specific keys, that an action remains associated with the account that created it even if another account signs in before the queued write runs, and that enqueue refuses to proceed without an authenticated owner. They also exercise coalescing behavior and removal of the legacy unscoped queue.

The queue processor tests model 401 responses and verify that the refused entry and unattempted suffix remain in order. They test the distinction between a credential failure and a permanently invalid client request. The application should not destroy a user's valid intent because a token has become invalid.

Credential tests go closer to the boundary. They use the real task service and API client request interceptor with a recording adapter, then record the method, URL, and authorization header for requests that reach the adapter. This lets a test assert which credential would have gone on the wire, or that no request was sent after the session changed. That is stronger than mocking the service and checking only whether it was called.

The server's session-invalidation suite describes a Mongo-independent model that mimics database snapshots and atomic conditional updates where races matter. It uses gates to force requests into specific interleavings, including password-reset races. Its comments note that a real MongoDB round trip lives in another integration suite requiring a downloaded mongod binary and therefore may not run in every environment. A deterministic model test can test a race, but it is not the same as executing the full integration test against MongoDB.

## What a test suite can and cannot prove

A test can demonstrate that a particular input and interleaving lead to an expected result under the test's assumptions. It does not prove the absence of all races, validate production environment variables, or establish that an external provider is available. A passing test count is a dated observation, not a permanent property of a repository.

The client and server manifests define verification scripts. The client runs Vitest and a production build. The server runs Vitest, source type-checking, test type-checking, and a build. The root script delegates to those package-level verification commands. This is a useful verification contract, but it is not evidence that the commands passed during this writing session. They were not executed for this book.

A stronger verification record would include the exact commit SHA, command, environment, result, and tests skipped because services were unavailable. That lets readers distinguish code review from executed verification and local verification from integration or production checks.

### Evidence trail

- [Queue tests](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-client/src/offline/syncQueue.test.ts)
- [Queue processor tests](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-client/src/offline/queueProcessor.test.ts)
- [Credential boundary tests](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-client/src/offline/queueProcessor.credentials.test.ts)
- [Session invalidation tests](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/tests/sessionInvalidation.test.ts)
- [Client verification scripts](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-client/package.json)
- [Server verification scripts](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/package.json)

### The next verification pass

Run the package verification commands against a clean checkout, record the commit and outcomes, then inspect gaps exposed by the results. Update the narrative if execution reveals failures, environment-dependent skips, or mismatches between test doubles and production behavior.
