# AquaVeda #157: When the DOM Was Ready Before the Test Observer

**Story status:** Evidence-backed engineering story  
**Repository:** [Mystify7777/aquaveda-v2](https://github.com/Mystify7777/aquaveda-v2)  
**Pull request:** [#157](https://github.com/Mystify7777/aquaveda-v2/pull/157)  
**Issue:** #55  
**Related fix:** [#158](https://github.com/Mystify7777/aquaveda-v2/pull/158)  
**Review date:** 2026-11-13

## The problem

PR #157 adds a public recommendation endpoint:

`GET /api/v1/issues/:issueId/recommendations`

The backend change is isolated to the recommendation route and its integration tests. Backend CI passed all 1,583 tests, and Playwright also passed.

The PR nevertheless failed the repository CI pipeline because the frontend test suite had one unrelated failure:

```text
src/components/expert/expert-status-provider.test.tsx
  ExpertStatusProvider: failure semantics
    no prior data: error (never presented as a status)

AssertionError: expected undefined to be 500

938 tests passed, 1 failed
```

The failing file was not changed by PR #157.

## The misleading part

The test was checking two different representations of the same state.

The `Probe` component renders the provider state into the DOM:

```text
user|error|-|-|fresh
```

It also publishes the complete provider value into an external `latest` variable from a React effect:

```text
render -> DOM update
       |
       +-> effect -> latest = value
```

The test did this:

```text
await waitFor(() => expect(text()).toBe("user|error|-|-|fresh"));
expect(latest.error?.status).toBe(500);
```

The first assertion proves that the rendered provider state reached `error`.

The second assertion immediately reads a separate observer that is updated by an effect.

That creates an unnecessary synchronization boundary. The CI failure showed exactly the dangerous state:

```text
DOM:    user|error|-|-|fresh
latest: error is still undefined
```

The product state was not necessarily wrong. The test's observation of that state was incomplete.

Humans have spent decades inventing increasingly sophisticated asynchronous runtimes and then testing them with a variable assigned in an effect. Civilization remains undefeated.

## Why this is a real test-design problem

Testing Library's `waitFor` is specifically intended to retry assertions until they pass, including asynchronous UI changes. React Testing Library also wraps `render` and related operations with React's `act` machinery. The important distinction here is that the test was waiting for the DOM representation, but not for its separate effect-driven observer to become synchronized with that representation. citeturn691289search8turn691289search9

The test therefore mixed two observation mechanisms:

1. **DOM observation** through `text()`;
2. **effect-published internal observation** through `latest`.

The first had already reached the expected state when the second was read.

The strongest evidence is the exact CI result: the rendered state assertion passed, while `latest.error?.status` was still `undefined`.

## The resolution

The fix in PR #158 keeps the production code unchanged and synchronizes the second observation explicitly:

```text
await waitFor(() => expect(text()).toBe("user|error|-|-|fresh"));
await waitFor(() => expect(latest.error?.status).toBe(500));
```

The important change is not the extra line itself. It is that each asynchronous observation now has its own completion condition.

The test no longer assumes:

> If the DOM changed, then every other observer of the same React state must already have updated.

Instead, it waits for the specific value it intends to inspect.

## What was not changed

The correction deliberately does not:

- modify `ExpertStatusProvider`;
- change error semantics;
- change API behavior;
- add delays;
- add arbitrary sleeps;
- weaken the assertion;
- remove the error-status check;
- alter PR #157's recommendation implementation.

This matters because the failure occurred in an existing frontend test while PR #157 touched only backend recommendation routing, documentation, and backend integration tests.

## Verification boundary

At the time of review, PR #157's CI state was:

| Verification | Result |
|---|---|
| Backend suite | Passed, 1,583/1,583 |
| Playwright | Passed |
| Frontend tests | Failed, 938/939 |
| Frontend lint/typecheck/build | Not reached because frontend tests failed |

The frontend failure was therefore a CI blocker, but not evidence against the recommendation API itself.

PR #158 contains the minimal test-only correction and is intended to restore the frontend verification gate before PR #157 is merged.

## The lesson

A test can fail even when the UI has already reached the expected state.

The subtle failure mode is:

```text
application state
      |
      +------> DOM observer       ✓ updated
      |
      +------> effect observer    ... not published yet
```

When a test observes asynchronous React state through multiple channels, each channel needs a clear synchronization point.

The useful question is not:

> "Did the state update?"

It is:

> **"Which observation of that state am I actually waiting for?"**

That distinction keeps a legitimate CI failure from turning into a pointless investigation of perfectly innocent production code.
