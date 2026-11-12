# AquaVeda #152: When the Wake-Up UX Became a Test-Synchronization Problem

**Story status:** Evidence-backed engineering story  
**Repository:** [Mystify7777/aquaveda-v2](https://github.com/Mystify7777/aquaveda-v2)  
**Pull request:** [#152](https://github.com/Mystify7777/aquaveda-v2/pull/152)  
**Issue:** #152  
**Review date:** 2026-11-12

## The problem

AquaVeda's public landing page depends on a backend that can become slow after inactivity. The product needed to distinguish ordinary loading from a request that had been waiting long enough to deserve an explicit wake-up message.

The intended behavior was simple:

- normal loading keeps the existing loading state;
- after three seconds, show a wake-up state;
- if the request succeeds, return to the real content;
- if it times out, show timeout-specific recovery;
- do not create a second request or pretend that the application knows a particular infrastructure event occurred.

The implementation was small. The test boundary was not.

## Why the tests kept failing

The commit history of the PR shows a sequence of corrective changes around the same underlying problem:

- timer-driven wake-up tests needed their React effects flushed before advancing fake time;
- the timeout rejection had to be attached before the fake clock was advanced;
- the wake-up timer had to be synchronized with React's effect lifecycle;
- the wake-up hook was isolated from the component to make the timing behavior deterministic;
- the landing-map integration initially needed correction so the wake-up state was actually rendered;
- the wake-up state also needed to reset correctly across loading transitions and re-entry.

The important point is that these were not separate product requirements. They were symptoms of one test-design problem:

> the test was advancing time faster than the component's asynchronous lifecycle had been allowed to settle.

A timer existing in the code does not mean React has already installed that timer at the instant the test reaches the next line. Humans invented fake clocks and then discovered that the rest of the runtime still exists. Magnificent species.

## The misleading failure

A naive test can look conceptually correct:

```text
render component
    |
advance fake timers by 3 seconds
    |
expect wake-up state
```

But the component's effect may not have run yet.

The actual sequence is closer to:

```text
render
  |
  +-- React schedules effect
          |
          +-- effect runs
                  |
                  +-- timer is registered
                          |
                          +-- 3 seconds elapse
                                  |
                                  +-- wake-up state updates
```

If the test advances the clock before the effect has been flushed, it can advance a clock for a timer that does not yet exist.

The resulting failure looks like a timing problem in the feature, when the real problem is that the test crossed an asynchronous boundary too early.

## The resolution

The PR gradually moved toward explicit synchronization:

1. **Let React effects settle before manipulating fake time.**
2. **Attach timeout rejection handlers before advancing the clock.**
3. **Keep the wake-up timing logic in a focused hook so its lifecycle can be tested independently.**
4. **Test the landing integration separately from the timer mechanism.**
5. **Verify that wake-up state resets when loading starts again.**
6. **Verify re-entry rather than testing only the first transition.**

This produced deterministic tests for:

- ordinary loading;
- the three-second wake-up transition;
- successful recovery after wake-up;
- API timeout classification;
- timeout-specific recovery copy;
- wake-up state reset and re-entry.

## The architectural correction

One important fix was not merely test-related.

The wake-up presentation had to be integrated at the actual landing-map request boundary:

```text
LandingMapPreview
        |
        +-- request starts
        |
        +-- normal loading
        |
        +-- > 3s
        |      |
        |      +-- WakeUpState
        |
        +-- success
        |      |
        |      +-- map content
        |
        +-- timeout/failure
               |
               +-- recovery state
```

The implementation deliberately did **not**:

- issue another request;
- implement polling;
- add automatic retry;
- claim that every slow request is definitely a Render cold start;
- promise a fixed recovery duration.

The three-second threshold is a UX signal, not an infrastructure detector.

## What made the final version robust

The final design separates three concerns:

### Request lifetime

The existing API timeout remains fifteen seconds.

### Wake-up presentation

A three-second threshold communicates that the request is taking longer than ordinary loading.

### Failure recovery

Actual timeout/failure remains a distinct state with its own recovery copy.

That separation prevents the UI from turning an uncertain infrastructure condition into a false claim.

## Verification

The final PR added focused deterministic coverage and passed the repository CI pipeline.

The merged change also preserved the existing API timeout and cancellation behavior rather than introducing a second async mechanism.

Final squash merge:

```text
2bede081870e7cd96a1a275d30d0902e600ca023
```

## The lesson

When a UI test involves **React effects + fake timers + asynchronous requests**, the order of operations is part of the test contract.

A reliable test does not merely advance time. It establishes that the system has reached the state in which the timer exists, then advances the clock, then flushes the resulting state transition.

The useful question is therefore not:

> "Why didn't three seconds make this component change?"

It is:

> **"Had the component actually reached the point where those three seconds mattered?"**

That distinction turned a fragile timing test into a deterministic one.
