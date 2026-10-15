# Chapter 2: Drawing the System's Boundaries

A reliable application begins by deciding which component is allowed to know what. HaloTask Pro is split into a browser client and an API server, but the more important boundaries exist inside those halves.

On the client, pages and components are separated from hooks, service wrappers, offline persistence, reminders, and Growth Tree logic. Dashboard components handle presentation; hooks coordinate state and lifecycle; service modules translate operations into API requests; offline modules persist cached data and queued actions. This layout gives responsibilities a home. It is not, by itself, proof that every runtime path respects those boundaries.

The server follows a similar division. Express routes connect endpoints to controllers, middleware establishes authentication and request boundaries, models represent persisted entities, and utility modules handle input validation, pagination, token parsing, tree rules, and process lifecycle. The task controller does not treat a task ID as proof of ownership: its database queries include both the task ID and authenticated user ID. Authentication establishes who is calling; the controller still has to scope the requested resource to that caller.

That distinction is foundational. Authentication and authorization answer different questions. A valid session can identify a user. It does not grant that user access to every task in the database. The task update and delete paths use user-scoped filters, so a valid bearer token alone is not enough to mutate another account's task.

## The browser is not the database

The IndexedDB wrapper in the client opens a database named halotasks-offline, currently at version 1, with a generic cache object store. It exposes get, set, and remove operations. Each stored value is accompanied by an update timestamp. Higher-level modules choose the keys and define what each value means.

This is a thin abstraction. The database wrapper does not know what a task, history snapshot, queue entry, or tree represents. That semantic responsibility belongs to the modules using it. It also means the generic database does not enforce every ownership rule. A module using a shared key can still create shared state; account scope must be designed into the storage contract.

The task cache uses simple keys for cached tasks and the last sync timestamp. The queue, task history, and Growth Tree use user-specific keys. The difference is consequential and should be reviewed as part of the full login/logout/load lifecycle, not judged from a key name in isolation.

## The server decides what becomes durable

The task API validates incoming bodies before persisting them. The controller checks required titles and field types, and translates Mongoose validation/cast failures into client errors instead of unexplained server failures. The server also owns the Growth Tree award path associated with task completion. That gives it a place to apply rules consistently instead of trusting the browser to award durable progress.

The client remains important: it must preserve unsent changes, explain pending state, and recover after interruptions. But it should not be allowed to rewrite the meaning of server-confirmed records merely because local state is convenient.

The flow can be summarized as:

**User action → client state → local persistence or queued intent → authenticated API request → validated server mutation → server response → client reconciliation**

Not every operation takes every step. An offline write may stop at local persistence until reconnection. The important point is that the system distinguishes an attempted action from an acknowledged one.

The README lists Vercel for the client and Render for the primary API, with Railway listed as a backup. These are documented deployment links, not proof of uptime or automatic failover. A backup URL is not a failover system unless routing and operational procedures actually make it one.

### Evidence trail

- [Client source tree](https://github.com/Mystify7777/Halotask-Pro/tree/main/halotasks-client/src)
- [Server task controller](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/src/controllers/task.controller.ts)
- [Authentication middleware](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/src/middleware/auth.middleware.ts)
- [Offline database wrapper](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-client/src/offline/db.ts)
- [Project README](https://github.com/Mystify7777/Halotask-Pro/blob/main/README.md)

### The boundary to keep watching

A module layout is evidence of intended separation, not proof that every path respects it. The next chapters follow the asynchronous edges, where responsibilities can leak if identity, ordering, or acknowledgement is not carried explicitly.
