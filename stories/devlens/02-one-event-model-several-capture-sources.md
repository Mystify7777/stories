# Chapter 2: One Event Model, Several Capture Sources

A shared event model is the point where unrelated browser signals become one product. Without it, the Panel would need separate renderers and query rules for runtime errors, console entries, network outcomes, and React component failures. DevLens instead defines a versioned `DevLensEvent` shape with identity, origin, category, severity, title, message, timestamp, and optional stack, metadata, context, and tags.

The model has two jobs. It gives the consumer a stable set of fields to render and query, and it gives capture packages a common contract they can satisfy without depending on one another.

## Normalize at the boundary

Each source has its own raw representation. A browser `ErrorEvent` is not shaped like a console call; a Fetch response is not shaped like an XHR abort; React provides a component stack through an error-boundary lifecycle. The capture package converts these inputs into the common event form before reporting them to Core.

That boundary is important because normalization should be local to the source that understands the raw data. Core should not need to import Fetch-specific classifiers or React types to understand an event. It can distribute and retain a normalized event without knowing the details of the original browser API.

The category model is extensible. Session-import validation checks that the category field exists and is a string, but it does not close the category values to a fixed list. This leaves room for plugins to introduce new categories while keeping the event's structural contract known.

## Identity, time, and immutability

The EventBus generates missing event identity and timestamps, applies middleware, then freezes the event before subscribers receive it. This establishes a useful invariant: consumers see a stable event rather than a mutable object whose meaning can change after it has been stored or rendered.

The event model also carries a version. Import validates that version rather than assuming any arbitrary JSON object is compatible with the current schema. A versioned event contract gives future changes a place to express compatibility, though versioning alone does not guarantee that old formats will remain importable forever.

## Deep freezing is a boundary defense

The implementation uses a deep-freeze helper rather than relying on TypeScript's compile-time readonly types alone. TypeScript types disappear at runtime, and JavaScript callers can still hold references to objects. Freezing the event prevents consumers from accidentally modifying its fields or nested metadata after publication.

The EventBus's middleware receives a mutable draft before the final event is frozen. That allows validation or enrichment middleware to transform an event during the reporting pipeline without exposing that mutability to subscribers. Middleware is synchronous and runs in registration order; it is intentionally not a general asynchronous workflow engine.

## Implementation trace: one completed Fetch request

The following trace follows the actual composition points at the pinned revision. It uses a completed Fetch operation as the example; a rejected request is classified through the rejection branch, while a fulfilled HTTP response is classified through the response branch.

1. **Capture settles.** The Fetch interceptor calls `handleFetchSettle(info)` after the operation settles. `createNetworkPlugin()` delegates outcome/severity classification, assembles a `CapturedRequest`, and passes it to the network normalizer.
2. **Normalize and report.** The network package's composition point is this exact line in `packages/network/src/network.ts`:

   ```ts
   bus.report(normalizeNetworkEvent(capturedRequest));
   ```

   Source: [network.ts at the pinned revision](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/network/src/network.ts).
3. **Complete the Bus boundary.** `EventBus.report()` fills missing `id`, `version`, and `timestamp`, runs synchronous middleware, deep-freezes the resulting event, pushes it into the replay buffer, and dispatches it to matching subscribers. The core implementation's final operations are:

   ```ts
   const finalEvent = deepFreeze(finalDraft, seen) as DevLensEvent;
   replayBuffer.push(finalEvent);
   dispatchToSubscribers(finalEvent);
   return finalEvent;
   ```

   Source: [event-bus.ts at the pinned revision](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/core/src/event-bus.ts). The `externallyOwned` input references are intentionally excluded from deep freezing; that exception is described below.
4. **Retain through explicit wiring.** `connectStoreToBus(bus, store)` subscribes to the Bus and forwards each event to `store.add(event)`. The Store is not implicitly coupled to the Bus. See [store-bus-connector.ts](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/core/src/store-bus-connector.ts).
5. **Inspect or export.** The Panel consumes the Store's retained history, while export serializes Store data. The Panel is not involved in capture and does not become a second event pipeline.

This trace is intentionally narrow: it identifies the real functions and boundaries without pretending that a network request, EventBus dispatch, Store notification, and UI render form one transaction.

## Deep freezing has a documented exception

The Bus deep-freezes the published event graph, but the input contract supports `externallyOwned` references that are excluded from that traversal. The marker itself is removed before the event draft is built; eligible object/function references are passed to the deep-freeze helper as exclusions. This lets callers preserve ownership of explicitly external references rather than having DevLens freeze them as a side effect. The ordinary event graph remains frozen, but the guarantee is not “every reachable object is always frozen.” See [event types](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/core/src/types.ts) and [EventBus implementation](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/core/src/event-bus.ts).

## The trade-off

A shared event schema makes downstream consumers simpler, but it can also tempt designers to flatten meaningful source differences. DevLens keeps source-specific detail in metadata/context while preserving common fields for severity, search, filtering, and display. That balance is only as good as each normalizer: if two sources encode the same concept differently, the shared Panel can still behave inconsistently.

The event model is therefore a public contract, not just a TypeScript interface. Changes should be additive where possible, tested across source packages, and recorded in an ADR when they alter architectural behavior.

**Sources:** [Core event types](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/core/src/types.ts), [EventBus](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/core/src/event-bus.ts), [deep-freeze helper](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/core/src/utils/deep-freeze.ts), [event model ADR](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0001-event-model.md).
