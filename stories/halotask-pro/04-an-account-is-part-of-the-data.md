# Chapter 4: An Account Is Part of the Data

A browser can outlive a login. A laptop can be shared. A password can be reset on another device while this one sits offline. These are ordinary events, but they expose a dangerous assumption in local-first applications: that data belongs to the browser because the browser happens to be holding it.

HaloTask Pro's queue implementation rejects that assumption. Queue keys are formed from the user ID. Reads and writes accept an explicit owner, and enqueue captures that owner before waiting for serialized work. Tests create two accounts with separate queues, switch between them, and verify that identical local task IDs do not cause one account's coalescing rules to alter the other's entries.

There is also a legacy problem. Before account-scoped queues, entries lived under one global key without ownership metadata. The implementation does not guess who created them. It removes the old key and discards entries whose owner cannot be proven. Some pending work may be lost, but assigning ambiguous mutations to the next person who logs in would be worse.

The same pattern appears in task history and Growth Tree storage. Both use user-specific keys. Old unscoped records are not adopted into the next account's state. The principle is simple: if ownership cannot be established, do not read, merge, or send the data as though ownership were known.

## Identity must survive asynchronous boundaries

A key can be correctly scoped and still be used incorrectly. Imagine a queued update created by account A. Replay starts. The request receives a 401 because a password reset revoked the token on another device. The response interceptor clears authentication. If the queue processor writes to “the current user's queue,” the current user may be nobody. If another person signs in quickly, it may be account B. The queue could be lost or reassigned even though the original key format was correct.

HaloTask Pro's processor binds the owner ID at the beginning and uses it for every queue read and write. It also binds the token to the same session. The API client refuses to send a request if the bound session no longer matches the current session. The result is a two-part identity rule: the queue's storage owner and the request credential must refer to the same session.

The request-level tests matter here. A storage test can prove that queue A is stored under key A. It cannot prove which bearer token ultimately reaches the network layer. The credential test uses the real task service and API request interceptor with a recording adapter, then asserts which authorization header reaches that adapter or that the request was never sent.

## Authentication state is versioned

The server middleware verifies the JWT, validates its payload, loads the user's current token version, and rejects a token if the version no longer matches. The authentication controller's comments explain why the token version comes from the same account read used during login: a login that races with a password reset must not issue a token carrying an obsolete generation.

The client cannot decide that a 401 means “throw the user's work away.” The server may be correctly refusing an old credential while the queued operation remains valid. Keeping the queue and revoking the session are compatible if the app preserves the action for the same account and never retries it as another account.

There is a privacy trade-off. User-scoped keys prevent application code from casually loading another account's records, but IndexedDB is not encrypted storage. This is logical isolation and correct routing, not cryptographic isolation against someone with access to the browser profile or developer tools.

### Evidence trail

- [Account-scoped sync queue](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-client/src/offline/syncQueue.ts)
- [Account-scoped task history](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-client/src/offline/history.ts)
- [Account-scoped Growth Tree storage](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-client/src/growth/treeStorage.ts)
- [Session-bound API requests](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-client/src/services/api.ts)
- [Server authentication middleware](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/src/middleware/auth.middleware.ts)
- [Session invalidation tests](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/tests/sessionInvalidation.test.ts)

### The rule this chapter leaves behind

Identity is not just an authentication header. It is part of the ownership of every local mutation, history snapshot, progression record, and asynchronous callback that may finish after the user has changed.
