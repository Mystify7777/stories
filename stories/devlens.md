# DevLens: A Diagnostics Engine Where Capture and Presentation Stay Separate

**Story status:** Evidence-backed draft; implementation claims are based on the public repository at `main`  
**Repository:** [Mystify7777/devlens](https://github.com/Mystify7777/devlens)  
**Review date:** 2026-10-15  
**Evidence boundary:** Read-only review of the README, workspace manifest, repository tree, selected architecture decision records, specifications, and GitHub Actions metadata. I did not clone the repository or run its build or test commands during this review.

## The problem

Browser diagnostics are often split across unrelated places: runtime exceptions in one stream, console output in another, network failures in DevTools, and framework-specific component errors elsewhere. A diagnostics overlay can bring those signals together, but only if the underlying event model is consistent and the capture mechanisms do not become entangled with the UI.

DevLens is a framework-agnostic, embeddable developer diagnostics engine for modern JavaScript applications. It normalizes several kinds of browser and component activity into one event stream and presents them in a live overlay. Its architecture is deliberately broader than the overlay itself: another consumer could eventually read the same event store without replacing the capture system.

The central design principle is **engine first, UI second**.

## Architecture: capture, distribute, retain, consume

The repository describes this pipeline:

```text
Runtime ─┐
Console ─┼─> EventBus ─> EventStore ─> Panel
Network ─┤
React   ─┘
```

The packages have separate responsibilities:

- **Core** defines the normalized event model, synchronous EventBus, EventStore, and plugin contract.
- **Runtime** observes browser runtime failures, including `window.error` and unhandled promise rejections.
- **Console** intercepts console methods while preserving the application's native console output.
- **Network** observes Fetch and asynchronous XMLHttpRequest operations and classifies their outcomes.
- **React** reports client-side errors caught by a React error boundary, including React's `componentStack`.
- **Panel** reads stored events and renders them in a Shadow-DOM-isolated overlay.
- **Playground** wires the packages together so capture paths can be exercised manually.

The separation is meaningful. Capture packages observe host-application behavior and report normalized events. The EventBus distributes them, the EventStore retains them, and the Panel presents them. The Panel does not capture events itself. This keeps UI decisions from dictating how a new source must observe the application.

## Design decisions that make the system more than a console overlay

### 1. A small, normalized event model

The event model is versioned and shared across capture sources. Runtime failures, console calls, and network outcomes can therefore be handled by common consumers without requiring each consumer to understand every browser API.

The EventBus is synchronous and dependency-free. The EventStore is intentionally separate from it and acts as a plain data structure. That separation also allows events from sources other than live capture, such as an imported session, to enter the Store without pretending they were freshly observed by a plugin.

### 2. Idempotent lifecycle for global capture plugins

Runtime, Console, and Network use a narrow `install()` / `uninstall()` contract, with both operations intended to be idempotent. Console and Network need to intercept existing APIs, so their implementations must preserve and restore original behavior rather than silently taking ownership of the host application.

Network makes this tradeoff explicit: it wraps Fetch and XMLHttpRequest because neither exposes the event-listener hook needed for this kind of observation. Uninstall restores the original references. The invasive mechanism is isolated inside the Network package instead of spreading into Core or Panel.

The React package does not force itself into that same interface. A React error boundary is a component managed by React's lifecycle, not a global listener or API patch. It reports through the shared EventBus and is explicitly client-only; it does not claim to capture React Server Component or server-rendering errors.

### 3. Network events represent completed operations

Network requests have a duration, unlike a runtime error or console call. The accepted Network ADR chooses to report one immutable event when a request settles, rather than emit a start/end pair or mutate a stored event in place.

This preserves the existing event and append-only Store model. It also establishes a known limitation: an operation that never settles, or a page torn down before settlement, may never produce a Network event. That is a documented tradeoff, not a capability to imply away.

The captured network record includes method, canonicalized URL, available HTTP status, duration, and an explicit outcome. Query-parameter values are redacted by default while parameter names are preserved. Request and response bodies and arbitrary headers are not captured. This conservative default matters because a network request may contain data the application never intended to expose in a diagnostic panel.

The documented outcome categories include success, HTTP error, network error, abort, timeout, and opaque responses. Fetch and XHR expose different signals, so some distinctions are best-effort rather than perfectly equivalent across both APIs.

### 4. React integration targets information that ordinary console capture can miss

The React integration is a client-only error boundary that reports caught errors through the same EventBus and includes React's `componentStack`. That stack identifies the component-tree location of a failure and is distinct from a JavaScript call stack.

The ADR records why this package is a capture mechanism rather than a wrapper around the Panel: it fills a specific observation gap and does not require the Panel to become React-dependent. A reactive adapter for the Panel's own UI state was considered but deferred because the repository had not established a real consumer need for that additional API.

### 5. Session import preserves the evidence being restored

The repository documents session import as complete. Its stated contract validates the imported session before adding it to an empty EventStore, preserves event IDs and timestamps, and maintains array order. It avoids adding a partially validated session to the live Store. This is useful for inspecting previously exported diagnostic sessions without inventing new identities or timestamps for old events.

The import path is intentionally narrow: merging an imported session into a Store that already contains live events is documented as out of scope. The feature should not be described as general-purpose session merging.

## Engineering process: decisions are recorded before they become accidental architecture

DevLens has a substantial set of Architecture Decision Records (ADRs), research notes, and feature specifications. The repository's documentation structure separates three questions:

- **ADRs:** Why was this architectural direction chosen?
- **Specifications:** What behavior and constraints does a feature need?
- **Implementation:** How does the code realize it?

That separation is backed by explicit amendments when a later decision changes earlier documentation. The React role ADR, for example, records contradictions in earlier docs and explains the chosen role instead of silently rewriting the history. The lint-policy ADR similarly records why a broken lint script was removed rather than adding a tool without a concrete need.

The project's engineering discipline is not simply “write lots of docs.” The useful part is that the docs record alternatives, rejected approaches, boundaries, and the evidence that would justify revisiting a decision. The repository also explicitly resists speculative abstractions: generalize when another real consumer justifies it, not merely because a helper or interface seems elegant.

## Current documented status

The README currently marks Core, Runtime, Console, Panel, Network, and React complete and tested, with the Playground working. It reports **879 test cases** across the suite and provides workspace commands for build, test, formatting, and TypeScript validation.

Those are repository claims, not a fresh test result from this review. The latest workflow metadata I inspected showed a successful Formatting Check run on 2026-09-25. I did not independently execute `pnpm test`, `pnpm build`, `pnpm typecheck`, or `pnpm format:check`, so this story does not assert that all commands pass on the current checkout.

The root manifest uses pnpm workspaces and defines `build`, `test`, `clean`, `format`, `format:check`, and `typecheck`. There is no dedicated lint script; the repository's ADR explains the current policy and identifies the remaining coverage gaps rather than claiming linting is universally unnecessary.

## Boundaries and limitations

The story should preserve the system's explicit non-goals:

- No request or response body capture, and no arbitrary header capture.
- No WebSocket, Server-Sent Events, or endpoint-grouping implementation in the Network package.
- No React Server Component or server-rendering error capture through the React boundary.
- No merging an imported session into a non-empty Store as part of the current import contract.
- No reactive React adapter for Panel UI state without evidence of a real consumer need.
- No claim that the reported 879 tests passed in this review; the count is taken from the README.

These boundaries strengthen the story. They show a system with explicit scope and tradeoffs rather than a fictional universal diagnostics solution.

## What to verify before finalizing this story

- Run the current workspace build, test, type-check, and formatting-check commands; record the actual results and environment.
- Confirm the README's test count against a current run.
- Inspect representative tests for Core, Runtime, Console, Network, React, Panel, and Session Import, including failure and cleanup paths.
- Confirm the latest CI state for every relevant workflow, not only formatting.
- Exercise the Playground end-to-end and verify the documented capture paths, network outcome labels, query redaction, and session restore behavior.
- Check that install/uninstall cycles preserve native browser APIs and that Console never suppresses native output.
- Verify the public API and package manifests match the documentation, including the distinction between the Plugin contract and the React error-boundary lifecycle.
- Keep future feature ideas clearly separate from implemented capabilities.

## Source trail

- [Repository README](https://github.com/Mystify7777/devlens/blob/main/README.md)
- [Design notes](https://github.com/Mystify7777/devlens/blob/main/DESIGN.md)
- [Documentation structure](https://github.com/Mystify7777/devlens/blob/main/docs/adr/README.md)
- [Event model ADR](https://github.com/Mystify7777/devlens/blob/main/docs/adr/0001-event-model.md)
- [Event Bus ADR](https://github.com/Mystify7777/devlens/blob/main/docs/adr/0002-event-bus.md)
- [Event Store ADR](https://github.com/Mystify7777/devlens/blob/main/docs/adr/0004-event-store.md)
- [Plugin contract ADR](https://github.com/Mystify7777/devlens/blob/main/docs/adr/0006-plugin-contract.md)
- [Panel ADR](https://github.com/Mystify7777/devlens/blob/main/docs/adr/0008-panel.md)
- [Network plugin ADR](https://github.com/Mystify7777/devlens/blob/main/docs/adr/0010-network-plugin.md)
- [Session Import ADR](https://github.com/Mystify7777/devlens/blob/main/docs/adr/0011-session-import.md)
- [React integration role ADR](https://github.com/Mystify7777/devlens/blob/main/docs/adr/0013-react-integration-role.md)
- [Lint policy ADR](https://github.com/Mystify7777/devlens/blob/main/docs/adr/0014-lint-policy.md)
- [Inspection specification](https://github.com/Mystify7777/devlens/blob/main/docs/specs/inspection.md)
- [Session Import specification](https://github.com/Mystify7777/devlens/blob/main/docs/specs/session-import.md)
- [Workspace package manifest](https://github.com/Mystify7777/devlens/blob/main/package.json)
- [GitHub Actions workflows](https://github.com/Mystify7777/devlens/actions)

## Summary

DevLens is best understood as a diagnostics engine with a replaceable presentation layer. Its strongest engineering story is the discipline of keeping capture, normalized events, storage, and UI separate while extending the same pipeline to harder cases such as network lifecycles, sensitive URL data, React component stacks, and imported sessions. The ADRs make the tradeoffs inspectable, and the explicit limits help keep the product honest. The next step is a fresh local verification pass before this draft is treated as final.
