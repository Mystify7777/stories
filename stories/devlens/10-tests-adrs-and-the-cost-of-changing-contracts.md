# Chapter 10: Tests, ADRs, and the Cost of Changing Contracts

DevLens has a substantial architecture record. Its ADRs do not merely announce decisions; several preserve alternatives that were considered, constraints that ruled them out, and amendments when later work exposed an overgeneralization. That history is especially useful in a library whose public event shape and package boundaries affect every integration.

## Decisions are recorded before they become invisible

The design principles state that public APIs should change slowly and that architectural decisions should be recorded before implementation locks in a shape. The ADRs trace the progression from the event model and Bus through Store, plugin lifecycle, Console, Panel, network capture, session import, Store capacity, and React integration.

The best examples document why a tempting shortcut was rejected. Network capture did not use mutable in-flight events because that would conflict with the immutable event model. Session import does not reuse the Bus because it would rewrite historical identity. React does not pretend to be a global plugin because its lifecycle is component-based. These are concrete constraints, not generic statements about clean architecture.

ADRs can also be amended. ADR-0013 corrects a broad implication in ADR-0006 about React integration. That is healthy decision maintenance: the record should preserve the original reasoning and clearly identify the correction rather than leave contradictory guidance for future contributors.

## Test structure mirrors the architecture

The repository has tests for Core event handling, ring-buffer behavior, plugin contracts, Runtime, Console normalization/interception, Network classifiers/interceptors/URL normalization, Panel rendering and controls, import/export, navigation, filtering, search, and React integration. The tests are organized close to the code and frequently isolate pure helpers from browser-facing behavior.

That organization makes failures easier to localize. A URL redaction failure should be diagnosable in a pure normalizer test before being investigated through the full Panel. A Store capacity failure should be testable without mounting the UI. The architecture and test seams reinforce each other.

However, test-file presence is not the same as a current green run. The README at the reviewed revision reports 879 test cases across the suite. A separate Session Import spec records 584/584 passing at its implementation milestone in August 2026. Those figures refer to different recorded points in the project's history and should not be blended into a single verified result. This documentation pass did not run tests, typecheck, build, or inspect a fresh CI result.

## A layered verification plan

Before treating this book as publication-ready, a fresh verification pass should record:

1. Exact Node and pnpm versions.
2. Install command and lockfile state.
3. Workspace test command, count, skips, and failures.
4. Typecheck and build results for all packages.
5. Formatting checks and the relevant CI workflow runs.
6. Browser-level checks for global hook restoration and host-app non-interference.
7. End-to-end Playground checks for capture, inspection, export, and import.

The browser-level tests are particularly important because mocked tests can verify wrapper logic without proving that a real browser's Fetch/XHR behavior remains unchanged across all edge cases.

## The engineering lesson

The ADRs and tests form a feedback loop. ADRs define invariants and explain why a design exists; tests make selected invariants executable; implementation experience can then expose a decision that needs amendment. Neither mechanism replaces the other. Tests cannot explain why a trade-off was accepted, and prose cannot prove that code still satisfies it.

**Sources:** [Engineering principles](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/DESIGN.md), [ADR index](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/README.md), [workspace manifest](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/package.json), [Session Import specification](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/specs/session-import.md), [GitHub Actions](https://github.com/Mystify7777/devlens/tree/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/.github/workflows).