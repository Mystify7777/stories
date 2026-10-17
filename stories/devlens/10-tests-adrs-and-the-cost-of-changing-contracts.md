# Chapter 10: Tests, ADRs, and the Cost of Changing Contracts

DevLens has a substantial architecture record. Its ADRs do not merely announce decisions; several preserve alternatives that were considered, constraints that ruled them out, and amendments when later work exposed an overgeneralization. That history is especially useful in a library whose public event shape and package boundaries affect every integration.

## Decisions are recorded before they become invisible

The design principles state that public APIs should change slowly and that architectural decisions should be recorded before implementation locks in a shape. The ADRs trace the progression from the event model and Bus through Store, plugin lifecycle, Console, Panel, network capture, session import, Store capacity, and React integration.

The best examples document why a tempting shortcut was rejected. Network capture did not use mutable in-flight events because that would conflict with the immutable event model. Session import does not reuse the Bus because it would rewrite historical identity. React does not pretend to be a global plugin because its lifecycle is component-based. These are concrete constraints, not generic statements about clean architecture.

ADRs can also be amended. ADR-0013 corrects a broad implication in ADR-0006 about React integration. That is healthy decision maintenance: the record should preserve the original reasoning and clearly identify the correction rather than leave contradictory guidance for future contributors.

## Case study: React did not fit the original plugin generalization

The original plugin-contract ADR grouped React with Console and Network and suggested the same `create*Plugin(bus): Plugin` factory shape for all of them. That was a reasonable-looking generalization, but it conflated two different lifecycle models: Console and Network install global hooks that need explicit restoration, while React capture is naturally attached to a component's mount/render lifecycle through an error boundary.

The contradiction became visible across the ADRs: ADR-0006 described React as a conventional capture plugin, ADR-0008 described a React wrapper around the Panel, and ADR-0009 used both framings. ADR-0013 resolved the conflict in favor of a client-only, error-boundary-based capture mechanism that reports React-specific `componentStack` data through the existing EventBus. The later README at the pinned revision describes `@devlens/react` as implemented; the ADR itself preserves the earlier decision point and should be read as historical evidence, not as a claim that the package was still absent at the reviewed revision.

The resulting lesson is specific: share the event-reporting contract without forcing every source into an identical lifecycle API. The correction preserves the earlier ADR wording as history and explicitly amends its implication instead of silently rewriting the decision trail. Sources: [ADR-0006](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0006-plugin-contract.md), [ADR-0008](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0008-panel.md), [ADR-0009](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0009-v0.3.0-direction.md), [ADR-0013](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0013-react-integration-role.md).

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

## Evidence matrix: three different levels of confidence

The following matrix records what this book can establish from the pinned source and what still requires execution. “Implemented” means the behavior is visible in the reviewed implementation. “Covered by tests” means relevant test files exist in the repository at the pinned revision; it does not claim those tests passed in this pass. “Verified in a fresh run” is reserved for checks actually executed against a recorded revision and environment.

| Claim | Implemented | Covered by tests in source tree | Verified in a fresh run for this book |
| --- | --- | --- | --- |
| Event model and immutability | Yes, with the documented `externallyOwned` exception | EventBus and deep-freeze test files are present | No |
| Capture lifecycle and native API restoration | Install/uninstall paths are implemented | Runtime, Console, Network lifecycle tests are present | No browser-level run performed |
| Network outcome classification and URL redaction | Classifiers and normalizer are implemented | Classifier/normalizer and interceptor tests are present | No |
| Panel filtering, search, keyboard, pause/resume | Inspection features are implemented | Focused Panel tests are present | No |
| Session import rejection before Store mutation | Validation precedes `addMany()` | Import validation and Store behavior tests are present | No |
| Workspace readiness | Repository describes packages as complete | README reports 879 test cases | No fresh test, typecheck, build, formatting, or CI result inspected |

The source paths and test inventory are the evidence for implementation and test coverage; current pass status remains unverified. Any later CI evidence should record the exact commit SHA, date, Node/pnpm versions, command, test totals, skips, and failures. A result for one revision must not silently be treated as a result for a later one.

## The engineering lesson

The ADRs and tests form a feedback loop. ADRs define invariants and explain why a design exists; tests make selected invariants executable; implementation experience can then expose a decision that needs amendment. Neither mechanism replaces the other. Tests cannot explain why a trade-off was accepted, and prose cannot prove that code still satisfies it.

**Sources:** [Engineering principles](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/DESIGN.md), [ADR index](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/README.md), [workspace manifest](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/package.json), [Session Import specification](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/specs/session-import.md), [GitHub Actions](https://github.com/Mystify7777/devlens/tree/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/.github/workflows).
