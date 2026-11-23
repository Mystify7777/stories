# AquaVeda #224: When Account Restoration Could Delete a Fresh Session

**Story status:** Evidence-backed engineering story  
**Repository:** [Mystify7777/aquaveda-v2](https://github.com/Mystify7777/aquaveda-v2)  
**Pull request:** [#224](https://github.com/Mystify7777/aquaveda-v2/pull/224)  
**Issue:** #219 (fixes #217)  
**Review date:** 2026-11-23

## The problem

AquaVeda's account-governance work introduced HARD_SUSPENDED accounts and several ways for a suspension to become active again:

- an Admin can lift the suspension;
- a request can discover that a temporary suspension has expired;
- login can discover the expiry;
- a background sweep can discover the expiry.

All of those paths can race with one another.

The dangerous part was not merely deciding which actor wins the state transition. The original restoration implementation also had to remove refresh sessions that were issued before the suspension.

That created a subtle race:

```text
stale expiry worker
      |
      | reads expired suspension
      | pauses
      v
Admin lifts suspension
      |
      v
account becomes ACTIVE
      |
      v
user logs in
      |
      v
fresh refresh session created
      |
      v
stale worker resumes
      |
      v
deletes sessions
```

The stale worker had already lost the state transition race, but its cleanup could still destroy the fresh credential that belonged to the restored account.

That is a security and availability bug hiding inside what looked like ordinary cleanup.

## Why simply moving the delete was not enough

The obvious fix was to move session deletion after the account-state update.

That removes one ordering problem but creates another.

Consider:

```text
restore account
      |
      v
ACTIVE state committed
      |
      X  process crashes
      |
      v
session cleanup never runs
```

The account is now active while old pre-suspension refresh credentials remain physically stored.

So the invariant could not safely be:

> "Restoration is secure if session deletion eventually succeeds."

The security property needed to survive a crash between the state transition and physical cleanup.

That changed the design.

## The key insight: credentials need an identity of generation

PR #224 introduced a credential-generation protocol.

Each user has an `authGeneration`.

Each refresh session issued under a non-zero generation records the same value as `Session.authGeneration`.

The refresh operation then enforces:

```text
session.authGeneration === user.authGeneration
```

If they differ, the refresh session is rejected.

Restoration increments the user's generation as part of the same conditional state transition that makes the account active.

The resulting sequence is:

```text
pre-suspension session
        |
        | generation 0
        v
HARD_SUSPENDED
        |
        | restoration commits
        | generation -> 1
        v
ACTIVE
        |
        +-- old session: generation 0 -> rejected
        |
        +-- new login -> session generation 1 -> usable
```

The important property is that **credential invalidation no longer depends on deleting the old session document**.

Physical cleanup becomes reconciliation rather than the security boundary.

## The race becomes safe

Return to the original race.

The stale worker reads an expired suspension and pauses.

An Admin wins the conditional restoration:

```text
accountState = ACTIVE
authGeneration = 1
```

The user logs in and receives a generation-1 refresh session.

The stale worker resumes.

Its conditional update fails because it lost the race.

It therefore performs no cleanup.

Even if a delayed cleanup from another path runs later, cleanup is bounded by the generation it was responsible for. A generation-0 cleanup cannot match a generation-1 session.

The stale worker can no longer destroy the user's fresh credential.

## The winner rule

The implementation uses one restoration core for:

- Admin lift;
- request-path expiry;
- login-path expiry;
- expiry sweep.

Only the actor that wins the conditional restoration update performs cleanup.

Cleanup is additionally bounded to sessions at or below the restored generation.

That gives the protocol two independent protections:

1. **State transition arbitration:** only one actor wins restoration.
2. **Credential-generation isolation:** stale cleanup cannot target credentials from a later generation.

This is stronger than trying to make every caller perfectly coordinate its cleanup timing.

## Cleanup is deliberately best-effort

Another important change is that cleanup failure no longer blocks restoration.

Suppose restoration succeeds but the database cleanup operation fails:

```text
ACTIVE + generation 1
        |
        +-- old session remains stored
        |
        +-- old session is unusable
```

The security invariant still holds because refresh checks the generation.

A later reconciliation can safely remove the stale document.

AquaVeda therefore provides:

- `purgeStaleSessions` as an idempotent reconciliation operation;
- login-time stale-session cleanup;
- the existing Session TTL as another eventual cleanup mechanism.

This turns cleanup into a maintenance concern instead of a credential-validity concern.

## The tests were designed around interleavings

The strongest part of the implementation is how the race was tested.

The tests do not use arbitrary sleeps such as:

```js
await sleep(100);
```

That would make the race probabilistic and the test flaky.

Instead, the implementation exposes inert governance test hooks and uses explicit barriers to stop an operation at a precise point.

That allows tests to construct scenarios such as:

```text
Worker A reads expired state
        |
        X  barrier
        |
Admin restores account
        |
User logs in
        |
Worker A resumes
        |
assert: Worker A deletes nothing
```

The test suite covers:

- stale expiry worker versus Admin lift;
- concurrent expiry workers;
- winner cleanup occurring after a fresh login;
- expiry versus login;
- Admin lift versus expiry;
- sweep versus Admin lift;
- delayed suspension-time cleanup;
- cleanup failure and recovery;
- pre-restoration refresh rejection;
- fresh post-restoration session usability;
- repeated credential generations;
- backward compatibility;
- MUTE isolation.

The tests also include mutation checks that deliberately reintroduce unsafe behavior and verify that the suite fails.

That is useful evidence because the tests are not merely proving the happy path. They are protecting the invariant that motivated the design.

## Backward compatibility was part of the protocol

The generation field is optional.

Legacy users and sessions without an `authGeneration` field continue to behave normally at generation zero.

Only non-zero generations are stored.

That means the protocol does not require a data migration merely to introduce the new credential-generation semantics.

After restoration, newly issued sessions carry the new generation.

## What this changed architecturally

The final design separates three concerns that had previously been too tightly coupled:

### Account state

Determines whether the account is ACTIVE or HARD_SUSPENDED.

### Credential validity

Determined by the user's current authentication generation.

### Physical session cleanup

Removes stale database records when convenient.

The relationship is now:

```text
account restoration
        |
        +--> atomically advance credential generation
        |
        +--> invalidate old credentials logically
        |
        +--> perform bounded best-effort cleanup
        |
        +--> reconcile leftovers later
```

That is the durable part of the fix.

## What remains unresolved

PR #224 deliberately did not solve every governance problem.

In particular:

- governance state transition and audit-event persistence remain separate writes;
- durable state/event atomicity remains the scope of #204;
- residual stale Session documents may remain until reconciliation or TTL;
- the test-hook seam is intentionally exposed for deterministic concurrency testing.

Those are explicit boundaries rather than hidden assumptions.

## Verification

The PR added deterministic concurrency coverage and was merged successfully.

Final squash merge:

```text
d2dc6d4e5ee83b974076a33ed1e6f28c411dc187
```

The implementation was verified against real MongoDB in the repository's validation process.

## The lesson

The interesting bug was not:

> "Two requests tried to restore the same account."

That race was already handled by the conditional state transition.

The real bug was:

> **A request that lost the state race could still perform side effects based on the stale state it had previously observed.**

That is a much broader concurrency pattern.

A stale worker can be harmless at the state-transition layer and still be dangerous if its cleanup is not tied to the version or generation of the state it is modifying.

The durable rule is:

> **When asynchronous cleanup can race with a state transition, do not make security depend on cleanup timing. Bind credentials and destructive side effects to an explicit generation or version.**

Then a stale worker can lose the race without becoming a destructive worker.

## Broader engineering takeaway

This pattern generalizes beyond account suspension.

The same reasoning applies whenever a system has:

- asynchronous workers;
- retries or delayed jobs;
- state transitions that create new credentials or resources;
- cleanup performed after those transitions;
- and the possibility that an older operation resumes after a newer state has been committed.

The safe structure is:

```text
state transition
      |
      v
new generation
      |
      +--> old credentials become invalid immediately
      |
      +--> stale workers become harmless
      |
      +--> cleanup becomes bounded reconciliation
```

The important engineering move was not adding another lock or hoping the delete happens in the right order.

It was changing the invariant so that **the wrong worker can resume without being able to damage the newer state**.

That is the part worth remembering.
