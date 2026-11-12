# Cold-Start Handling Across AquaVeda, HaloTask Pro, CyberShield, and StockSphere

**Story status:** Evidence-backed comparative engineering reference  
**Repositories:** [AquaVeda v2](https://github.com/Mystify7777/aquaveda-v2), [HaloTask Pro](https://github.com/Mystify7777/Halotask-Pro), [CyberShield](https://github.com/Mystify7777/CyberShield), [StockSphere](https://github.com/Mystify7777/StockSphere)  
**Review date:** 2026-11-12

## Why this comparison exists

Several of these applications are deployed on infrastructure where the backend can be asleep or slow to become responsive. The implementation pattern evolved across the projects.

This story records that evolution so future projects have a concrete reference instead of rediscovering the same problem through another collection of loading spinners and disappointed users.

The important distinction is that **"cold-start handling" is not one problem**. There are at least three separate concerns:

1. **Detection:** How does the client know the backend is slow, unavailable, or actually ready?
2. **Presentation:** What does the user see while the backend is becoming usable?
3. **Failure semantics:** What happens when the backend never becomes usable?

The four projects make different trade-offs in these areas.

---

## Executive comparison

| Project | Detection model | User-facing behavior | Failure handling | Overall assessment |
| --- | --- | --- | --- | --- |
| **AquaVeda v2** | Request-duration threshold; 3s presentation threshold, 15s hard request timeout | Replaces normal map loading with an explicit wake-up state after the request remains pending | Existing API error path takes over after the bounded request timeout | **Best simplicity and UX separation** |
| **HaloTask Pro** | Real public health endpoint, sequential probes, 90s wake budget, unhealthy streak detection | Reuses cold-start overlay before login/register; can dismiss it without blocking the form | Distinguishes timeout/unreachable from backend unhealthy; manual retry | **Most technically complete readiness model** |
| **CyberShield** | Proactive startup warm-up requests to API health and AI service, each bounded at 15s | Full-screen "Waking up CyberShield" overlay while warm-up is pending | Warm-up failures are intentionally silent; app shell proceeds after completion/fallback | **Good proactive warm-up, weak readiness semantics** |
| **StockSphere** | Public health endpoint checked on auth landing and again before auth submission; 8s per probe | Server status, warming message, retry button, delayed hint after 10s | Failed health check leaves auth available, but there is no sustained polling/wake budget | **Useful first-generation UX, but shallow recovery model** |

The ranking depends on what is being optimized.

- **Best end-user UX architecture:** AquaVeda.
- **Best backend readiness model:** HaloTask Pro.
- **Best lightweight proactive warm-up:** CyberShield.
- **Best simple baseline:** StockSphere.

AquaVeda should therefore not be described as universally "more advanced" than HaloTask. HaloTask has substantially richer readiness detection. AquaVeda is better because it deliberately avoids turning a simple slow request into a miniature infrastructure-monitoring subsystem.

---

# 1. AquaVeda v2

## Implementation

AquaVeda's current landing map makes the cold-start problem local to the actual request that matters.

The map begins with:

- `issues === null`
- no error
- a real `getIssues()` request in flight.

The `useApiWakeUpState` hook watches that loading lifecycle. After **3 seconds**, it changes presentation from the ordinary skeleton/loading state to `WakeUpState`.

The important detail is that the 3-second threshold is explicitly documented as a **UX threshold**, not a claim that the backend infrastructure is definitely cold.

The API client separately enforces a **15-second hard request timeout**. That timeout covers connection, headers, and body reading, and aborts the underlying request rather than merely abandoning the promise.

Relevant implementation:

- `src/hooks/use-api-wake-up-state.ts`
- `src/components/landing/landing-map-preview.tsx`
- `src/lib/api/client.ts`

## What it gets right

### 1. It does not pretend a timer can prove infrastructure state

The client does not say:

> "The server is cold."

Instead, it says, in effect:

> "This request has been slow enough that the user deserves an explanation."

That is a much safer abstraction.

A browser cannot reliably distinguish every cause of a slow first request. It may be a sleeping host, network latency, overloaded infrastructure, a database connection delay, or another transport condition.

### 2. Presentation and request lifetime are separate

There are two different clocks:

- **3s:** change the UI explanation.
- **15s:** stop waiting for the request.

That separation is excellent.

A slow request does not automatically become a special infrastructure state, and the UI does not need to know deployment-specific details to remain honest.

### 3. The wake-up state is integrated into the real product surface

The actual landing map renders:

`WakeUpState` when the request is still pending and the UX threshold has elapsed.

This avoids a disconnected global "server waking" screen when only one part of the page is actually waiting for data.

### 4. The request remains bounded

The API boundary guarantees a finite lifetime. A cold or busy API cannot leave authenticated UI hanging forever.

The timeout is also classified separately through `isApiTimeout()`, while caller cancellation is distinguished from failure.

## What it does not do

AquaVeda does **not** proactively ask the backend whether it is ready.

It also does not:

- poll a health endpoint,
- retry automatically,
- maintain a 90-second wake budget,
- distinguish "backend unhealthy" from "host never answered" at the infrastructure level.

That is intentional and appropriate for this surface.

The landing map already has a real request whose success or failure determines whether the data is available. Adding a second readiness system would increase complexity without necessarily improving the user's outcome.

## Verdict

**AquaVeda is the cleanest cold-start UX pattern of the four when the problem is a slow data request on an otherwise usable page.**

Its strongest idea is not the wake-up message itself. It is the separation:

**request lifecycle → UX threshold → hard timeout → existing error recovery**

That is the pattern worth carrying forward.

---

# 2. HaloTask Pro

## Implementation

HaloTask Pro takes the problem much further.

It exposes a public `GET /api/health` endpoint and uses a dedicated `useBackendReadiness` hook before authentication.

The client distinguishes:

- `checking`
- `waking`
- `ready`
- `error`
- `idle` while offline.

The readiness probe distinguishes:

- `ok`
- `unhealthy`
- `no-answer`

The current implementation uses:

- **1.5s** slow-probe threshold before showing warm-up UI
- **2s** delay between sequential probes
- **10s** individual probe timeout
- **90s** total wake budget
- **3 consecutive unhealthy responses** before declaring the backend unhealthy

The warm-up is available on login and registration without requiring a session.

Relevant implementation:

- `halotasks-server/src/routes/health.routes.ts`
- `halotasks-client/src/services/healthService.ts`
- `halotasks-client/src/hooks/useBackendReadiness.ts`
- `halotasks-client/src/components/BackendWarmup.tsx`

## What it gets right

This is the strongest implementation if the goal is **actual readiness detection**.

### 1. It does not use a timer to declare readiness

Only an actual healthy probe produces `ready`.

That is materially stronger than simply waiting 15 or 30 seconds and assuming the backend has probably woken up.

### 2. It distinguishes failure classes

The user can get a different outcome for:

- no healthy response within the wake budget
- a backend that is responding but repeatedly reports that its database is unavailable.

That distinction is operationally useful.

### 3. It works before authentication

The readiness endpoint requires no session and does not load protected data.

That solves a common chicken-and-egg problem:

> the login page cannot wait for authenticated API state because the user is not authenticated yet.

### 4. It handles browser connectivity

The hook reacts to `online` and `offline` events rather than blindly continuing to poll while the browser has no network.

### 5. It provides explicit retry semantics

After a terminal failure, the user can ask for another check.

## Costs

HaloTask's approach is considerably more complex than AquaVeda's.

It introduces:

- a dedicated readiness endpoint,
- a readiness service,
- a state machine,
- polling,
- multiple timers,
- failure classification,
- offline handling,
- a wake budget,
- unhealthy streak accounting,
- additional tests and synchronization concerns.

That complexity is justified when **pre-auth backend readiness itself is a product concern**.

It would be excessive if all that is needed is:

> "This one public widget is taking a few seconds longer than usual."

## Verdict

**HaloTask Pro has the strongest infrastructure/readiness model, but AquaVeda has the cleaner product-level model.**

This distinction matters. The lesson is not to replace HaloTask with AquaVeda everywhere. The lesson is to choose the smallest model that matches the actual problem.

---

# 3. CyberShield

## Implementation

CyberShield uses a proactive warm-up strategy at application startup.

The root application starts two requests:

- `GET /system/health`
- the configured AI service base URL.

Both are sent through a `warmUrl()` helper using:

- `GET`
- `no-cors`
- `no-store`
- a **15-second AbortController timeout**.

The application displays a full-screen overlay:

> "Waking up CyberShield"

while the warm-up requests are pending.

The app shell becomes usable when both warm-up promises settle, with a separate 15-second fallback timer.

Relevant implementation:

- `client/src/App.jsx`
- `client/src/pages/ai/ScamDetector.jsx`
- `server/src/routes/systemRoutes.js`

The server exposes:

- `GET /api/system/health`
- `GET /api/system/version`
- `GET /api/system/uptime`

## What it gets right

### 1. It starts warming before the user needs the backend

This is a sensible deployment optimization.

The application does not wait until the first protected operation before attempting to wake hosted services.

### 2. It includes the AI dependency

CyberShield is not only dependent on its main API. The AI service is part of the product's actual runtime path.

Warming both dependencies recognizes that reality.

### 3. It keeps the warm-up bounded

The 15-second timeout prevents startup from becoming an infinite wait.

### 4. It provides domain-specific recovery guidance

The Scam Detector can recognize an AI-service failure and tell the user that the server may be waking up, with a retry action.

## Where it falls short

### 1. The warm-up is not really a readiness contract

The warm-up request is essentially:

> "Try talking to this URL."

It does not establish the same strong readiness semantics as HaloTask.

In particular, the client does not consume a structured readiness result from the warm-up and maintain a meaningful `ready/unhealthy/timeout` state.

### 2. `no-cors` deliberately limits what the browser can observe

The request is useful as a network warm-up, but the browser cannot inspect a normal cross-origin response body under `no-cors`.

That makes this fundamentally different from HaloTask's explicit health-probe contract.

### 3. The overlay can become a global tax

A startup warm-up affects the whole application.

Even if the user only wants to browse a public route that does not need the AI service, the application initially presents the same global waking state.

### 4. Failure semantics are intentionally vague

Warm-up failures are swallowed.

That is defensible for a best-effort optimization, but it means the system is not really telling the user:

- what is unavailable,
- whether the backend is ready,
- whether the AI service is ready,
- or whether the startup attempt actually succeeded.

## Verdict

**CyberShield has a good warm-up idea but weak readiness semantics.**

It is useful when the goal is:

> "Start waking hosted dependencies as early as possible."

It is less suitable as a reference for:

> "Tell the user accurately whether the service is ready."

---

# 4. StockSphere

## Implementation

StockSphere uses a public `GET /api/health` endpoint.

The auth landing page performs an initial health check with an **8-second request timeout**.

The UI exposes:

- `Waking server…`
- `Server ready`
- `Backend waking up (free hosting)...`
- `Retry health check`
- a delayed "Still waking up" hint after **10 seconds**.

When the user submits login/register while the server is not ready, StockSphere sets a warming state, displays a toast explaining that the free-tier backend may take 30–60 seconds, then performs another health check before continuing with authentication.

Relevant implementation:

- `frontend/src/App.jsx`
- `backend/src/main/java/com/stocksphere/common/controller/HealthController.java`

## What it gets right

### 1. The backend has a dedicated health endpoint

This is a clean foundation.

### 2. The UI explicitly acknowledges cold starts

Users are not left staring at a generic spinner.

### 3. There is a manual retry path

The user can ask the client to check again.

### 4. It keeps authentication available

The health check does not permanently block the auth form.

## Where it falls short

### 1. It only performs a single bounded check

An 8-second timeout is much shorter than the documented 30–60 second cold-start expectation.

The implementation does not establish a sustained wake budget.

### 2. The user can still be caught at the wrong moment

The strongest cold-start path is triggered when the user submits authentication.

That means the cold-start experience is partly reactive rather than being a reliable background readiness process.

### 3. The "30–60 seconds" message is not derived from actual readiness

It is explanatory copy, not measured state.

### 4. No explicit unhealthy state exists

A failed health request is broadly treated as "not ready."

The client does not distinguish:

- sleeping host,
- network failure,
- backend failure,
- database failure,
- or another HTTP failure.

## Verdict

**StockSphere is a useful baseline, but it is the least complete of the four readiness models.**

Its strongest lesson is simply that a dedicated health endpoint plus explicit UI feedback is substantially better than pretending the first API request will always be fast.

---

# What evolved across the projects

The progression is useful because it shows four different answers to the same deployment problem.

### StockSphere

**Health check + status UI + manual retry**

This established the basic pattern.

### CyberShield

**Proactive dependency warm-up**

The system began waking infrastructure before the user explicitly needed it.

### HaloTask Pro

**Real readiness state machine**

The health check became an actual readiness contract with polling, failure classification, offline handling, and a wake budget.

### AquaVeda v2

**Request-local UX threshold + bounded API lifetime**

The solution became smaller again.

That last step is important. More machinery is not automatically better engineering.

---

# Recommended pattern for future projects

Do not copy any one implementation blindly.

Use this decision tree.

## Case A: A page is waiting on one real API request

Prefer the **AquaVeda pattern**.

1. Start the real request.
2. Render normal loading UI.
3. After roughly 2–3 seconds, change the explanation to a wake-up/slow-service state.
4. Keep the request bounded.
5. On timeout, show a specific recovery state.
6. Do not claim that the infrastructure is definitely cold.
7. Do not add a separate health endpoint unless the product actually needs infrastructure readiness.

This should be the default.

## Case B: The entire application needs backend readiness before use

Use the **HaloTask pattern**, but only when justified.

1. Provide a public, minimal readiness endpoint.
2. Define an explicit success contract.
3. Distinguish no-answer from unhealthy backend responses.
4. Use sequential probes rather than overlapping polling.
5. Define a maximum wake budget.
6. Provide manual retry.
7. Handle browser offline/online transitions.
8. Keep readiness UI independent from authentication state.

This is appropriate for applications where backend readiness is genuinely a prerequisite.

## Case C: Multiple hosted dependencies need warming

Use the **CyberShield idea**, but improve the semantics.

1. Warm dependencies proactively.
2. Give each dependency an explicit readiness contract where possible.
3. Do not use `no-cors` as a substitute for observable health when the browser needs to know the result.
4. Do not globally block unrelated UI unless those dependencies are actually required.
5. Distinguish best-effort warm-up from readiness.

## Case D: Simple application with a small backend

Use the **StockSphere baseline** only if the requirements are genuinely simple.

A health endpoint plus retry is enough for many small applications, but do not advertise a 30–60 second recovery window while giving the browser only one 8-second probe.

---

# The reusable design rule

The most important lesson from these projects is:

> **Do not make the UI pretend to know more about infrastructure than the client can actually observe.**

A frontend can reliably know:

- a request is pending,
- a request has exceeded a UX threshold,
- a request succeeded,
- a request failed,
- a request timed out,
- a health endpoint returned a defined readiness response.

It usually cannot reliably know:

- "Render is waking up right now,"
- "MongoDB is still connecting,"
- "the container will be ready in 20 seconds."

Those are infrastructure hypotheses unless the backend exposes them as real state.

Therefore:

**Use timers to change the explanation, not to invent infrastructure facts.**

**Use health probes when readiness itself matters.**

**Use hard request timeouts to bound waiting.**

**Use explicit recovery states when failure is distinguishable.**

And, because apparently humans need the same lesson repeated across four repositories before it becomes doctrine, **do not ship a generic spinner and call the problem solved.**

---

# Reference implementation hierarchy

For future projects, the preferred hierarchy is:

```
                Does the product actually need
                backend readiness as a state?
                         /          \
                       no            yes
                       /              \
          Use request-local       Use explicit
          slow-state UX           readiness contract
                 |                       |
             AquaVeda              HaloTask Pro
                 |                       |
        + bounded timeout       + structured probe
        + honest messaging      + failure classification
        + existing error path   + wake budget
        + no fake readiness     + retry/offline handling

        Optional:
        proactive dependency warming
        when multiple services justify it
                 |
            CyberShield
```

StockSphere remains a useful simpler reference for a health endpoint and manual retry, but it should not be copied as the final model when cold starts can exceed a single probe's timeout.

---

# Final assessment

The projects are not a simple progression from "bad" to "good."

They represent different engineering trade-offs:

- **StockSphere:** straightforward but shallow.
- **CyberShield:** proactive but opaque.
- **HaloTask Pro:** technically rigorous but comparatively complex.
- **AquaVeda v2:** deliberately narrow, honest, and well integrated into the actual user-facing request lifecycle.

So the claim that **AquaVeda currently has the better cold-start handling** is defensible if "better" means **better product UX for a slow request with less unnecessary infrastructure machinery**.

It is not defensible if "better" means **more sophisticated backend readiness detection**. HaloTask Pro wins that category.

The reusable lesson for future projects is therefore not "copy AquaVeda."

It is:

> **Start with AquaVeda's small model. Escalate toward HaloTask's readiness model only when the product actually requires readiness as a first-class state.**

That gives future projects a reference point without turning every deployment delay into a distributed-systems thesis.
