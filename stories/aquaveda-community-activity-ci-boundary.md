# AquaVeda #186: When CI Found the Bug in the Test, Not the Feature

**Story status:** Evidence-backed engineering story  
**Repository:** [Mystify7777/aquaveda-v2](https://github.com/Mystify7777/aquaveda-v2)  
**Pull request:** [#186](https://github.com/Mystify7777/aquaveda-v2/pull/186)  
**Issue:** #166  
**Review date:** 2026-11-15

## The problem

AquaVeda needed a unified public activity feed for community-facing content.

The activity projection had to combine three different source types:

- Discussions;
- Issues;
- Knowledge entries.

The projection also had to respect each source's visibility rules, derive activity from the latest public comment where comments existed, provide comment counts, order activity deterministically, and paginate the merged result at the database level.

That sounds like a normal aggregation feature until three different resource types, several visibility contracts, and pagination all decide to coexist in one query. Software has a special talent for turning "show the latest activity" into a small distributed treaty.

PR #186 implemented:

- `GET /api/v1/community/activity`;
- a single MongoDB aggregation across the eligible source sets;
- database-side latest-comment selection and comment counts;
- database-side pagination and total calculation;
- batched actor hydration after pagination;
- explicit public DTO projection;
- deterministic ordering by `activityAt DESC`, then `sourceType ASC`, then `resourceObjectId DESC`;
- service, route, validation, and visibility tests.

## The first CI failure

The implementation itself was not what CI rejected.

The first verification run, **CI #176**, completed with:

- frontend tests passing;
- lint passing;
- TypeScript passing;
- production build passing;
- browser smoke tests passing;
- **1,648 backend tests passing and 1 failing**.

The single failure was in:

`server/tests/community-activity.service.test.js`

The failing case checked deterministic ordering when two activity records shared the same timestamp and therefore required the secondary `sourceType` and tertiary `resourceObjectId` tie-breakers.

At first glance, that looked like a possible aggregation defect.

It wasn't.

## The important clue was the runtime value

The aggregation returned `activityAt` as a native JavaScript `Date`.

That was the expected service-layer representation.

The test, however, converted the returned value to an ISO string:

```js
timestamp.toISOString()
```

and then compared that string directly with the returned value.

Conceptually, the test was asking:

```text
Date object === ISO string
```

Those are different runtime types even when they represent the exact same instant.

The failure therefore came from the test's assumption about the service contract, not from the activity ordering implementation.

## Separating the service boundary from the HTTP boundary

The key distinction was the serialization boundary.

Inside the service:

```text
MongoDB aggregation
        |
        v
JavaScript Date
        |
        v
service result
```

At the HTTP boundary:

```text
service result
        |
        v
Express JSON serialization
        |
        v
ISO timestamp in JSON
```

The test had accidentally pulled the HTTP representation into a service-layer assertion.

That would have been easy to "fix" in the wrong direction by changing production code to stringify dates earlier.

That would also have made the service less natural for its own consumers merely to satisfy an incorrect assertion.

## The minimal fix

The production implementation was left unchanged.

Instead, the test was corrected to compare the timestamps by their actual instant:

```js
assert.equal(
  new Date(result.items[0].activityAt).getTime(),
  timestamp.getTime(),
);

assert.equal(
  new Date(result.items[1].activityAt).getTime(),
  timestamp.getTime(),
);
```

This preserves the service-layer contract while still proving the behavior that matters:

> both activity records carry the expected timestamp, and their ordering is determined by the specified tie-breakers.

The fix was deliberately test-only.

No production serialization was introduced.

No aggregation logic was rewritten.

No weaker assertion was added simply to make CI green.

## Why this mattered

There are two different questions a test can ask:

1. **Does the service return the correct timestamp value?**
2. **Does the HTTP API serialize that timestamp into the expected JSON representation?**

Those are both legitimate tests.

They just belong at different boundaries.

The service test should verify the semantic value.

The route/API test should verify the serialized representation.

Mixing those boundaries creates tests that can fail even when the implementation is behaving correctly.

## The debugging path

The useful engineering sequence was:

```text
Community Activity implementation
        |
        v
CI #176 fails
        |
        v
1 backend test / 1,648 backend tests passed
        |
        v
Locate exact failing assertion
        |
        v
Inspect actual aggregation result
        |
        v
activityAt is a native Date
        |
        v
Compare against test expectation
        |
        v
Test expected ISO string
        |
        v
Identify service/API boundary mismatch
        |
        v
Test-only semantic timestamp comparison
        |
        v
CI #177 passes
```

This is the part worth preserving.

The failure was small, but the debugging method was correct: inspect the actual runtime value before changing production code.

## Verification

After the test-only correction, **CI #177** completed successfully.

The final PR head was:

```
a1cb319c0514ebaf73cbc1438a37f724de1285fb
```

The corrective commit was:

```
a1cb319c0514ebaf73cbc1438a37f724de1285fb
test: compare activity timestamps by instant
```

The PR was subsequently squash-merged into `main` as:

```
0ae3094d1538692805207134c772426ca5e12f53
```

## The larger feature story

The timestamp test failure was only one verification step in a larger backend feature.

The completed Community Activity projection established a single public activity surface across:

```text
Discussion
    |
    +-- active only
    +-- may exist without comments
    +-- latest public comment updates activityAt

Issue
    |
    +-- existing public visibility contract
    +-- requires public comment activity

Knowledge
    |
    +-- approved only
    +-- requires public comment activity

          |
          v

Unified Activity Projection
          |
          +-- activityAt
          +-- commentCount
          +-- actor
          +-- source metadata
          |
          v
Deterministic global ordering
          |
          v
Database-side pagination
          |
          v
Public API
```

The implementation deliberately avoided:

- a persisted Activity entity;
- an application-memory merge of every source record;
- a per-item request-time query loop;
- exposing moderation history or private fields.

The public DTO was explicitly allowlisted, and actor hydration was batched after pagination.

## The lesson

A red CI check is evidence of a mismatch somewhere.

It is not evidence that production code is guilty.

In this case, the implementation correctly returned a native `Date`. The test incorrectly treated the service result as though it were already an HTTP JSON payload.

The durable debugging rule is:

> **Before changing production code to satisfy a failing test, inspect the actual runtime value and identify which layer owns the representation being asserted.**

That prevents a common failure mode:

```text
wrong test assumption
        |
        v
production workaround
        |
        v
"green" CI
        |
        v
weaker architecture
```

The better path is:

```text
failing test
        |
        v
inspect behavior
        |
        v
identify contract boundary
        |
        v
fix the assertion at the correct layer
        |
        v
keep production semantics intact
```

## Broader engineering takeaway

PR #186 is useful evidence of a broader engineering habit: **treat CI failures as diagnostic information, not instructions to edit the nearest production line.**

The feature still needed to satisfy visibility, aggregation, pagination, ordering, actor hydration, and API-boundary contracts.

The test failure added another contract question:

> What representation should this layer actually own?

The answer was simple:

- MongoDB/service layer: semantic time value;
- HTTP layer: serialized timestamp;
- tests: assert the contract of the layer they are testing.

That distinction kept the fix small, preserved the implementation, and let the final system pass the full verification pipeline.

