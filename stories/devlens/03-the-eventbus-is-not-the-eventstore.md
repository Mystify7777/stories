# Chapter 3: The EventBus Is Not the EventStore

The EventBus and EventStore both hold events, which makes it easy to assume they are duplicate abstractions. Their histories serve different purposes.

The Bus keeps a small, bounded replay buffer so late subscribers can catch up on recent events and callers can inspect recent dispatch history. The Store is the longer-lived, queryable log used by the Panel and other consumers. At the reviewed source revision, the documented defaults are 1,000 events for Bus replay and 10,000 for the Store.

## Distribution versus retention

The EventBus handles reporting, ID/timestamp assignment when absent, middleware, category or wildcard subscriptions, optional replay, and dispatch. It is synchronous by design. Runtime and console events are naturally produced synchronously, and async plugins can schedule their own work rather than forcing every dispatch to become asynchronous.

The Store has a narrower role: add events, clear, read all, query by category, filter, subscribe, and destroy. It does not know where events originate. Its wiring to the Bus is explicit through `connectStoreToBus()`, rather than being hidden inside the Store constructor. This lets the same Store accept events from another source, such as an imported session, without changing the data structure.

## Why two buffers?

The Bus replay buffer exists to preserve dispatch continuity for late subscribers. It is not the canonical full history. The Store exists to support the user-facing diagnostics experience and has a larger retention capacity.

Clearing the Store does not clear the Bus replay buffer. That is intentional separation, but it creates a semantic edge that must be understood: a newly connected Store that replays old Bus history can receive events that were cleared from a previous Store. The two buffers are independent and are not presented as one transactional log.

Both use a ring buffer to keep bounded insertion efficient. A ring buffer avoids repeatedly shifting an array when the oldest item must be discarded. Once full, retention is bounded: old entries are evicted as new ones arrive. DevLens is a live diagnostics tool, not a permanent audit log.

## The connector keeps dependencies explicit

The Store no longer accepts an EventBus in its constructor. Instead, the connector subscribes the Store to the Bus and returns lifecycle wiring that can be managed independently. This makes the data structure easier to test and avoids giving the Store unnecessary knowledge about its source.

The Store's `addMany()` operation, introduced for session import, appends a batch and notifies subscribers once after the batch is inserted. It does not validate or freeze the events itself. Those responsibilities belong to the import boundary, which validates and freezes the complete batch before mutating storage.

## Capacity is a contract

The later retention-size ADR adds explicit `capacity` and `size` properties. Capacity is the configured maximum for the Store's lifetime; size is the number of events currently retained. Size rises as events arrive and stops at capacity when the buffer is full. It is not a lifetime event counter.

These distinctions matter for import: if a session contains more events than the target Store can retain, ordinary ring-buffer insertion would silently evict some imported events. The import spec therefore rejects oversized sessions before mutation. A successful import must preserve the entire session rather than report success after silently dropping part of it.

**Sources:** [EventBus implementation](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/core/src/event-bus.ts), [EventStore implementation](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/core/src/store.ts), [Store-Bus connector](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/core/src/store-bus-connector.ts), [EventBus ADR](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0002-event-bus.md), [EventStore ADR](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0004-event-store.md), [retention size ADR](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0012-eventstore-size.md).