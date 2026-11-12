# Engineering Decisions and Evidence Timeline

This page is a navigation index for the major architectural decisions visible at [source revision `f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef`](https://github.com/Mystify7777/devlens/commit/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef), not a second narrative of every decision. See [Chapter 10](10-tests-adrs-and-the-cost-of-changing-contracts.md) for the React correction case study and verification matrix; the linked ADRs remain the primary decision record.

## Decision index

- **One versioned event model:** every capture source reports through the same `DevLensEvent` contract.
- **Synchronous EventBus:** middleware and subscriptions run synchronously; async work is opt-in at the source.
- **Separate Store:** retention/querying is distinct from dispatch and replay.
- **Bounded buffers:** Bus replay and Store retention have different capacities and purposes.
- **Idempotent global capture lifecycle:** install/uninstall must not duplicate hooks or leave monkey-patches behind.
- **Panel in Shadow DOM:** the overlay is isolated from host CSS and uses direct DOM rendering.
- **Completed-operation network events:** one event is reported when Fetch/XHR settles, not a mutable start/end pair.
- **Privacy-aware URL normalization:** canonicalize URLs and redact non-empty query values by default.
- **Session import bypasses Bus:** validate and freeze a complete session, then insert directly into an empty Store while preserving IDs, timestamps, and order.
- **React is component-shaped:** React integration reports through the Bus without pretending to be a global plugin.
- **ADR amendments are allowed:** later evidence can correct earlier generalizations while preserving the decision history.

## Selected evidence trail

| Topic | Primary source |
| --- | --- |
| Event model | [ADR-0001](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0001-event-model.md) |
| EventBus | [ADR-0002](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0002-event-bus.md) |
| EventStore | [ADR-0004](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0004-event-store.md) |
| Plugin contract | [ADR-0006](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0006-plugin-contract.md) |
| Panel | [ADR-0008](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0008-panel.md) |
| v0.3 direction and inspection | [ADR-0009](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0009-v0.3.0-direction.md) |
| Network | [ADR-0010](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0010-network-plugin.md) |
| Session import | [ADR-0011](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0011-session-import.md) |
| Store size | [ADR-0012](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0012-eventstore-size.md) |
| React integration | [ADR-0013](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0013-react-integration-role.md) |

## Evidence limits and open verification work

The pinned repository README reports 879 test cases and describes the packages as complete. This documentation pass did not run tests, typecheck, build, formatting checks, browser scenarios, or inspect a fresh CI result. The Session Import spec's 584/584 result is a historical milestone from August 2026, not the current workspace count.

The most important verification areas are:
- confirm uninstall restores original browser APIs and listeners under repeated lifecycle calls;
- test Fetch/XHR edge cases in real browsers, including abort, timeout, opaque responses, redirects, and body-consumption behavior;
- verify URL redaction on unusual and malformed inputs;
- verify Panel isolation, keyboard navigation, filtering/search composition, and pause/resume semantics;
- verify invalid imports leave the Store untouched and preserve event identity/order on successful round trips;
- run all package tests, typecheck, build, formatting, and current CI workflows.

The book intentionally distinguishes implementation from intent. Research documents record investigated options; ADRs record accepted decisions; specs record concrete contracts; source files show implementation; tests provide bounded evidence. None alone proves every behavior in a deployed host application.
