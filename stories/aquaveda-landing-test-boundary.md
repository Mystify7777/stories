# AquaVeda #151: When a Correct `try/catch` Still Made the Test Fail

**Story status:** Evidence-backed engineering story  
**Repository:** [Mystify7777/aquaveda-v2](https://github.com/Mystify7777/aquaveda-v2)  
**Pull request:** [#151](https://github.com/Mystify7777/aquaveda-v2/pull/151)  
**Issue:** #150  
**Review date:** 2026-11-11

## The problem

AquaVeda's landing page was being rebuilt from a development-status screen into a real product entry point. Part of that work introduced a small public issue-map preview.

The preview had a deliberately important failure contract:

- request a bounded number of public issues;
- do not block the landing page;
- show real public data when available;
- show an honest empty state when there are no issues;
- and remain usable when the issue service fails.

The production component already implemented the failure path with an ordinary asynchronous `try/catch`:

```text
loadIssues()
    |
    +-- await getIssues(...)
    |
    +-- success -> setIssues(...)
    |
    +-- failure -> setError(...)
```

There was no missing error handler in the component.

The problem appeared only in the test for that failure path.

## The misleading failure

The test originally made `getIssues()` reject:

```ts
mockGetIssues.mockRejectedValue(new Error("offline"));
```

Vitest reported the error at the line where the test created the rejection:

```text
Error: offline
  landing-map-preview.test.tsx:75
  throw new Error("offline");
```

That looked suspicious because the component was supposed to catch exactly that failure.

Several variations were tried:

- an immediately rejected mock;
- `async () => { throw ... }`;
- a manually deferred Promise;
- rejecting the Promise inside `act()`.

The production component remained unchanged because its `try/catch` was structurally correct.

The key distinction was eventually identified: the test was intentionally creating a rejected Promise, and Vitest's unhandled-rejection machinery could observe that rejection at the test-controlled Promise boundary even though the application's asynchronous code subsequently handled the failure.

The stack location was therefore misleading. It identified where the rejection was created, not necessarily where application error handling failed.

## The better test boundary

The test did not actually need to test JavaScript's Promise rejection machinery.

It needed to test this application behavior:

> When reading the returned issue data fails, the component catches the failure and renders its fallback.

So the final test changed the failure mechanism.

Instead of returning a rejected Promise, the mock returns a successfully resolved object whose `items` property throws when the component accesses it:

```ts
mockGetIssues.mockResolvedValue({
  get items(): Issue[] {
    throw failure;
  },
  page: 1,
  limit: 8,
  total: 0,
  totalPages: 0,
});
```

The production code then executes:

```ts
const result = await getIssues(...);
setIssues(result.items);
```

The `await` succeeds. The access to `result.items` throws. Crucially, that access happens **inside the existing `try/catch`**.

So the test now exercises the exact error-handling boundary without creating an unhandled rejected Promise:

```text
getIssues()
    |
    +-- Promise resolves
            |
            +-- result.items
                    |
                    +-- throws "offline"
                            |
                            +-- component catch
                                    |
                                    +-- setError(...)
```

This was the actual patch that made the test pass.

## Why this was the right fix

The final solution did not weaken the production error path or add test-only behavior to the component.

It changed the test from:

> "Can this component survive a rejected Promise without Vitest reporting the rejection?"

to:

> "Does this component handle an exception thrown while processing the successful response?"

The second question is much closer to the contract the test actually needed to establish.

It also avoids coupling the test to the test runner's global unhandled-rejection detection.

## The lesson

There are two different things that can look like "the error path":

1. **Promise rejection handling**
2. **Synchronous exception handling inside an async function**

They are related, but they are not the same test boundary.

When the production code already owns the Promise rejection with a `try/catch`, repeatedly changing how the mock rejects the Promise can turn into a fight with the test runner rather than a test of the application.

The better question is:

> **What exact operation inside the application's error boundary should fail?**

Once that was identified, the test became deterministic and the production code required no change.

## What did not change

The production implementation retained:

```ts
try {
  const result = await getIssues({ page: 1, limit: PREVIEW_LIMIT });
  if (active) setIssues(result.items);
} catch (reason: unknown) {
  if (active) {
    setError(
      reason instanceof ApiError
        ? reason
        : new ApiError("Could not load issues", "network"),
    );
  }
}
```

The `active` cleanup guard also remained intact.

No production workaround was introduced.

## Verification

The final PR head was:

```text
d9d51ac0a249b18444601b52be677acddabe470a
```

CI completed successfully on workflow run #105.

The PR was merged by squash merge as:

```text
b7b5a728e5fa3902095b5be56681f6fbaebe0d35
```

The landing-page work therefore entered `main` with the failure-path test passing and the production error handling unchanged.

## The broader engineering takeaway

A failing test does not automatically mean the production code is wrong.

Sometimes the failure is in the **test's model of the boundary**.

In this case, the component was already doing the responsible thing: catching the failure and rendering a graceful fallback. The test was accidentally asking Vitest to judge an intentionally rejected Promise at the same time it was asking React to handle that rejection.

The final fix was not more error handling.

It was a more precise test.

That distinction matters because otherwise a harmless test-runner interaction can tempt engineers into changing correct production code just to make a red CI badge turn green. Software development has enough superstition already.
