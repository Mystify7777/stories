# Chapter 7: Inspection Is a Set of Independent State Transformations

A useful diagnostics panel needs more than a list. Developers need to narrow the event set, search messages, inspect metadata and stack traces, navigate without a mouse, pause automatic refresh, and export the current evidence. The risk is turning the Panel into a tightly coupled state machine where every new control changes the meaning of the others.

The inspection specification takes a compositional approach: the event list remains the primary navigation surface, while selection, filters, search, and inspection are separate views over the Store's data.

## Filtering has explicit algebra

Filter state is Panel-local. It is not written into the Store and does not mutate events. Categories within the category dimension combine with OR; severities within the severity dimension combine with OR; the dimensions combine with AND. An empty array means “no constraint on this dimension,” not “match nothing.”

This gives a precise invariant: applying category and severity filters in either order produces the same result as applying both together. The implementation is a pure function that preserves the relative order of matching events. A pure function is easier to test because it has no DOM, rendering, selection, or side-effect dependencies.

## Search complements rather than replaces filters

Search covers event text such as titles, messages, stack information, and relevant tags/metadata according to the implemented search contract. It should operate over the currently relevant event collection without changing the underlying Store. Highlighting is presentation, not a rewrite of the source event.

Keeping search and filters independent matters because their composition should be predictable. A developer can filter to network errors and then search for a route fragment without one control unexpectedly resetting another. The spec records interaction rules and invariants rather than leaving behavior to whichever order event handlers happen to run.

## Selection and keyboard navigation

The event list remains the primary navigation surface. Selection identifies an event for the inspector, but the inspector does not own event data or become a second navigation model. Keyboard navigation makes the list usable without requiring every action to be a pointer click; its behavior must remain consistent as filtering changes the visible list.

That introduces edge cases worth testing: what happens when the selected event disappears under a filter, when the list becomes empty, or when new events arrive while selection is active? These are state-transition questions, not merely accessibility polish.

## Pause does not mean stop capturing

Pause/resume controls freeze the Panel's automatic refresh. They do not uninstall capture plugins or stop the EventBus from receiving events. This distinction preserves the capture pipeline while letting a developer inspect a stable view. When refresh resumes, the Panel can catch up with the Store's current state.

Clear is another scoped action: it clears the Store rather than implying that browser capture has been disabled. The Bus's replay buffer is separate, so the exact behavior after clearing and reconnecting must follow the documented Bus/Store semantics.

## Export is a snapshot; import is a separate operation

Export serializes stored events into JSON. It should not change the Store, and it should preserve the event fields that make the snapshot useful for later inspection. Import has stricter semantics because the file is external input and can be malformed or malicious. The next chapter examines that trust boundary in detail.

**Sources:** [Inspection specification](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/specs/inspection.md), [filters](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/panel/src/filters.ts), [search](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/panel/src/search.ts), [navigation](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/panel/src/navigation.ts), [serialization](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/panel/src/serialize.ts).