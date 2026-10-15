# Chapter 6: A Tree That Must Not Lie

The Growth Tree gives HaloTask Pro its most visible metaphor. Complete tasks, gain experience, develop a streak, and see a tree change over time. A visual reward can make progress tangible, but it introduces a difficult question: who is allowed to decide that progress happened?

The implementation assigns that authority to the server. Growth Tree storage describes the server as the durable source of truth for the current user's tree. IndexedDB holds a user-scoped copy so an offline start has something to display, but that local copy is a fallback, not a state the client pushes back to the server. When the server is reachable, initialization adopts the server tree as-is. Offline completions are described as an in-memory preview; the queued task mutation later asks the server to award progress, and the response replaces the preview.

That design rejects a tempting merge rule: “whichever device has more XP wins.” A higher local XP count might be stale, duplicated, or generated under assumptions the server has since rejected. Merging by maximum value would reward optimism, not necessarily correctness. Instead, the server computes awards and the client renders the result it receives.

## Completion is a domain operation

The task controller's completion path invokes server-side award logic and can include a Growth Tree result in the task response. Completing a task is therefore more than flipping a boolean. It is a domain operation with related consequences: persist completion, award progression according to server rules, and return the resulting state so the client can reconcile its display.

Server-side tree rules and award helpers live separately from the controller. This gives the application a place to define streak calculations, stage changes, XP rules, and idempotency around completion. The repository includes tests for tree rules and awards, including a Mongoose-command suite. Their presence is evidence that the behavior is being tested at several layers, but this writing pass did not execute those tests.

The offline queue connects this system to account isolation. When a queued create or update is replayed, the processor checks for a Growth Tree result and forwards it only if the original session is still active. It avoids applying a response from one account to the UI state of another account that signed in while the request was in flight.

## A metaphor with a consistency contract

The tree's emotional job is to make effort visible. Its engineering job is to avoid making a promise the rest of the system cannot support. A local animation can be immediate, but durable XP should follow a server-confirmed completion. The storage module describes ignoring a response whose XP is older than the current value, because bulk requests can finish out of order.

The source also describes a legacy migration boundary: unscoped tree records in IndexedDB or localStorage have no provable owner and are deleted rather than adopted. A motivational feature gets no exemption from data ownership rules just because its state looks harmless.

### Evidence trail

- [Growth Tree storage and authority model](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-client/src/growth/treeStorage.ts)
- [Tree progression logic](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-client/src/growth/treeLogic.ts)
- [Server award logic](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/src/utils/treeAward.ts)
- [Server tree rules](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/src/utils/treeRules.ts)
- [Tree rule tests](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/tests/treeRules.test.ts)
- [Tree award tests](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/tests/treeAward.test.ts)

### What this design does not solve

A server-authoritative model is only as dependable as its transaction and idempotency behavior. Those details should be established from the award implementation and persistence tests before making broad claims about every concurrent completion scenario. The narrower supported claim is that the client is designed to adopt server-confirmed progression rather than upload its own optimistic XP total.
