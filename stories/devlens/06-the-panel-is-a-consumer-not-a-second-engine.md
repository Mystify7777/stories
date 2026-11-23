# Chapter 6: The Panel Is a Consumer, Not a Second Engine

The Panel exists to make captured events useful to a developer. Its original milestone was intentionally narrow: mount a floating overlay, subscribe to the Store, and render events as they arrive. Search, filters, inspection, docking, draggable positioning, keyboard shortcuts, and virtualization were deferred at first rather than being smuggled into the initial scope.

That restraint mattered. The architecture first had to prove that the Panel could consume the Store without owning capture or inventing another event model.

## Shadow DOM is a product requirement

DevLens is intended to be embedded in arbitrary host applications. A normal element appended to the document body remains exposed to the host application's global CSS: resets, broad selectors, or framework styles can unexpectedly change the overlay. Shadow DOM provides a boundary in both directions. Host styles do not casually reach into the panel, and the panel's styles do not leak out into the application.

The implementation creates a host element and attaches an open shadow root. The panel uses vanilla TypeScript and DOM APIs rather than requiring a rendering framework. This preserves the ability to embed DevLens into applications with different UI stacks.

## Incremental rendering, not a virtual DOM

The Panel updates DOM nodes directly as Store state changes. It appends event rows and removes older rows when the visible limit is exceeded. The ADR explicitly avoids calling this virtual-DOM diffing: there is no tree reconciliation algorithm. The strategy is deliberately simple because the goal is a lightweight diagnostic overlay that appears with minimal configuration.

The trade-off is that direct DOM updates require careful ownership of nodes, subscriptions, and teardown. The Panel must unsubscribe from the Store and remove its host element on uninstall. Otherwise repeated mounting can leave stale DOM or duplicate listeners behind.

## Display limits and retained history differ

The Store may retain thousands of events, but the Panel has its own visible-row limit. That separates the amount of history available for queries and export from the amount rendered at once. A display cap can keep the UI manageable, but it should not be mistaken for Store retention capacity or for a guarantee of permanent history.

## The UI must reflect the Store

The Panel is not allowed to become a second source of truth. Inspection reads the selected event; filtering and search derive a subset; export serializes stored events; import validates and adds history to the Store. These operations should not silently mutate event content or create a separate event database inside the renderer.

This also makes alternative consumers possible. If a CLI or editor extension reads the Store in the future, it should reuse the same normalized events without depending on the Panel's DOM or state model.

**Sources:** [Panel ADR](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0008-panel.md), [Panel implementation](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/panel/src/panel.ts), [overlay implementation](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/panel/src/overlay.ts), [renderer](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/panel/src/renderer.ts).
