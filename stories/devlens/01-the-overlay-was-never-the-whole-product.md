# Chapter 1: The Overlay Was Never the Whole Product

A developer diagnostics panel looks like a UI problem: collect some errors, put them in a list, and make the list readable. The difficult part begins before the first row appears. Runtime exceptions, console calls, network requests, and framework errors originate through different browser APIs, have different lifecycles, and expose different metadata. If each feature feeds a bespoke UI path, the product becomes a pile of special cases rather than a diagnostics system.

DevLens takes the opposite direction. Its engineering principles state: **engine first, UI second**. The engine defines a shared event model and pipeline; the Panel is one consumer. A future CLI or editor extension should be able to read the same Store without changing capture packages.

## The pipeline is the product boundary

The system is divided into packages with different responsibilities:

- `@devlens/core` owns event types, the synchronous EventBus, EventStore, bounded ring buffer, and the common plugin contract.
- `@devlens/runtime` observes uncaught browser errors and unhandled promise rejections.
- `@devlens/console` intercepts console calls and converts them into normalized events.
- `@devlens/network` observes Fetch and asynchronous XMLHttpRequest operations.
- `@devlens/react` reports errors caught by a client-side React error boundary.
- `@devlens/panel` renders events from the Store and provides inspection, navigation, search, filtering, pause/resume, export, and import.
- The Playground wires the packages together for manual exploration.

This separation is not merely folder organization. It establishes a dependency direction: capture sources report through Core's public contract, while the Panel consumes stored events. The Panel should not need to know how a browser error was captured, and a capture plugin should not need to know how the Panel renders it.

## Framework-agnostic means a boundary, not a slogan

The core package is designed to avoid framework dependencies. React support is an integration package, not a dependency that leaks into Core. The same idea applies to the Panel: it uses vanilla TypeScript and DOM APIs rather than requiring the host application to use React or another rendering framework.

The practical goal is a diagnostics overlay that can be embedded into an existing application without asking the application to adopt DevLens's UI stack. Shadow DOM isolates the overlay's styles from the host page and keeps the host's global CSS from unexpectedly restyling the panel.

## The architecture grows by preserving invariants

The repository's ADR history shows a sequence of increasing complexity: a shared event model, EventBus, Store, plugin contract, Panel, inspection, network capture, session import, retention-size semantics, and React integration. New capabilities were designed against the existing contracts rather than being allowed to quietly redefine them.

That approach creates a useful engineering test: when a new feature appears, does it fit the event pipeline, or is it trying to create a second source of truth? Network capture, for example, must represent completed requests without mutating existing events. Session import must preserve old event identity, which makes the normal EventBus reporting path unsuitable. React integration observes component errors but does not need global install/uninstall behavior.

## What this book will examine

The chapters that follow focus on the trade-offs hidden behind those boundaries: why the Bus and Store are distinct, how global interception is made reversible, how network outcomes are classified, why query values are redacted, how the Panel's independent state transformations compose, and why imported history is validated before it enters storage.

**Evidence boundary:** This is based on the public repository at [pinned revision](https://github.com/Mystify7777/devlens/commit/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef). The repository README describes the packages as complete and tested, but this documentation pass did not run the suite.

**Sources:** [README](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/README.md), [DESIGN.md](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/DESIGN.md), [ADR index](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/README.md).
