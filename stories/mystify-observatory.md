# Mystify Observatory: Building a Portfolio That Explains Its Own Engineering

**Story status:** First evidence-backed draft  
**Repository inspected:** [Mystify7777/Mystify](https://github.com/Mystify7777/Mystify)  
**Review date:** 2026-10-13  
**Evidence boundary:** Read-only inspection of the public README, package manifest, selected source files, tests, and architecture documents. No local build or test run was performed for this story.

## The problem

A conventional portfolio usually presents finished projects and a list of technologies. Mystify Observatory takes a different approach: it treats the portfolio itself as a software system, with multiple ways to inspect the same underlying work.

The project's own framing is a portfolio operating environment, not an operating-system clone. Its three intended visitor modes are:

- **Recruiter:** concise, decision-oriented evidence.
- **Explorer:** discovery and environmental storytelling.
- **Engineer:** implementation detail and observability.

The governing design principle is that presentation may vary, but the underlying facts should not. The architecture documents express this as *Category Integrity*: different kinds of truth remain separate so each can stay honest in its own domain.

## Architectural intent

The planning record is unusually explicit. The repository contains a Master Context document, twelve constitutional documents, an ontology and relationship catalog, registry and schema specifications, invariant and governance documents, session specifications, and locked wireframes.

The architectural documents distinguish several concepts that should not be collapsed into one generic record:

- **Observations:** recorded facts about what was witnessed.
- **Discoveries:** interpretations or findings derived from evidence.
- **Patterns:** recurring conclusions.
- **Constraints and tradeoffs:** separate records rather than prose buried inside project cards.
- **Relationships:** explicit links that express how records connect.

The invariant specification also defines an authority boundary: only the Validation Engine should prevent a graph mutation. Consistency checks, derivation, and activation have separate responsibilities. These are documented design rules; this review does not establish that the complete graph engine is already implemented.

## What the current source demonstrates

The repository is a Next.js App Router application written in TypeScript. Its package manifest lists React, Zustand, Framer Motion, Tailwind CSS, Vitest, ESLint, Prettier, and TypeScript.

The source is organized into application entry points, shared shell components, a registry data layer, feature-specific UI, utilities, and state. The homepage renders the ObservatoryShell.

### 1. A shared shell with selectable modes

The shell includes a mode selector, dock, command palette, notifications, status bar, window manager, and surface primitive. The mode selector exposes Recruiter, Explorer, and Engineer as one active mode, supports arrow-key navigation, and persists the selected mode in session storage through a Zustand store.

The window manager reads window state from that store and supports focusing, minimizing, and closing open windows. These are real source-level behaviors, not just wireframe descriptions.

### 2. A staged boot sequence

The repository now contains a dedicated Observatory boot feature. Its state machine progresses through INITIALIZING, MORPHING, TRACING, GREETING, and COMPLETE. Stage durations are explicit, reduced-motion users receive shorter transitions, and scheduled transitions return a cleanup function.

A test file checks ordered progression, terminal completion, reduced-motion timing, cancellation, and one-time scheduled execution. The existence of these tests is verified; they were not executed during this read-only review.

The shell also contains a separate Recruiter boot sequence. The two sequences should not be conflated when comparing implementation with the locked wireframes.

### 3. A typed registry layer

The registry defines typed records for projects, constraints, tradeoffs, observations, patterns, and discoveries. The store exposes accessors and counts over imported registry data, including lookups by ID and project-related filtering.

The Recruiter content module derives boot text and quick signals from registry counts, and defines identity, credential, contact, and engineering-identity content. This is a concrete example of the project's stated principle that presentation should read from structured data rather than invent its own facts.

### 4. Tests around behavior and content contracts

The repository includes tests for the Observatory boot state, the Zustand state store, registry storage, and Recruiter content. These tests indicate intended contracts around mode persistence, invalid stored values, boot progression, and displayed content. Test presence should not be confused with a passing current CI run.

## Important distinction: design record versus shipped behavior

The design documentation is much broader than the current source surface. The code inspected in this pass establishes a working shell foundation, a mode selector, boot sequences, a typed registry, and Recruiter-specific content. It does **not** by itself prove that the full Explorer and Engineer experiences, a mutable ontology graph, validation/consistency/derivation engines, or every wireframe interaction are implemented.

That gap is not necessarily a defect: the project README describes the repository as a foundation, and the planning record explicitly sequences implementation. The accurate story is about a system being built from explicit architectural constraints, not a claim that every designed subsystem already ships.

## Engineering decisions worth explaining

1. **One environment, multiple lenses.** Modes change what is surfaced; they are not separate products with separate sources of truth.
2. **Structured domain records.** Projects and engineering evidence have typed representations that can feed different presentation surfaces.
3. **Explicit state boundaries.** The mode and shell state are centralized in a Zustand store; session persistence is isolated behind helper functions.
4. **Timed UI as a state machine.** Boot progression is represented as named stages with testable transitions rather than a single opaque animation delay.
5. **Reduced-motion support.** The boot sequence uses Framer Motion's reduced-motion preference and shortens stage durations.
6. **Documented invariants before implementation.** The repository records intended authority and consistency rules before the complete engine exists.

## Verification still required

Before this story is treated as final, verify the following against the current branch and CI:

- Run the declared test, type-check, lint, formatting, and production-build commands.
- Confirm the live boot sequence matches the locked wireframe, including timing, transitions, reduced-motion behavior, and completion semantics.
- Inspect the complete registry data and verify every displayed count is derived from the intended canonical records.
- Trace each mode from selection through its actual content. Do not imply Explorer or Engineer features exist solely because their modes are selectable.
- Compare the constitution, ontology, registry, invariants, and schema specifications with the runtime implementation. Label unimplemented design as planned.
- Verify resume and contact paths, accessibility behavior, keyboard interaction, responsive layouts, and failure states.
- Reconcile repository README status language with the actual implementation stage.

## Source trail

- [Repository README](https://github.com/Mystify7777/Mystify/blob/master/README.md)
- [Master Context v2](https://github.com/Mystify7777/Mystify/blob/master/Mystify_Observatory_Master_Context_v2.md)
- [Ontology and relationship catalog](https://github.com/Mystify7777/Mystify/blob/master/Observatory%20Core%20Docs/Observatory/observatory-doc1-ontology.md)
- [Invariant specification](https://github.com/Mystify7777/Mystify/blob/master/Observatory%20Core%20Docs/Observatory/observatory-doc3-invariants.md)
- [Application entry point](https://github.com/Mystify7777/Mystify/blob/master/src/app/page.tsx)
- [Observatory shell](https://github.com/Mystify7777/Mystify/blob/master/src/components/shell/observatory-shell.tsx)
- [Boot state machine](https://github.com/Mystify7777/Mystify/blob/master/src/features/observatory-boot/boot-state.ts)
- [Boot state tests](https://github.com/Mystify7777/Mystify/blob/master/src/features/observatory-boot/boot-state.test.ts)
- [Registry types](https://github.com/Mystify7777/Mystify/blob/master/src/data/registry/types.ts)
- [Registry store](https://github.com/Mystify7777/Mystify/blob/master/src/data/registry/store.ts)
- [Observatory state store](https://github.com/Mystify7777/Mystify/blob/master/src/state/use-observatory-store.ts)
- [Package manifest](https://github.com/Mystify7777/Mystify/blob/master/package.json)

## Summary

Mystify Observatory's most distinctive engineering idea is not the desktop metaphor. It is the attempt to make a portfolio's evidence, relationships, constraints, and decisions first-class data, then let different visitors inspect that data through different interfaces. The implementation already contains meaningful foundations for that idea, while the full architecture remains a larger target that must be described honestly as it is built.
