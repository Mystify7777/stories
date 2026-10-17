# Chapter 13: The Product Can Be Correct in One Service and Still Fail

CyberShield runs across a React/Vite client, an Express API, MongoDB, and a separate FastAPI service. The frontend is deployed to Vercel and the backend and AI service are documented as separate Render services. That separation lets components evolve and deploy independently, but it also introduces failure modes that cannot be fixed inside one repository folder.

## API configuration is a product contract

The client uses a configured API base URL, while the backend calls the AI service through `AI_SERVICE_URL`. These values must point to reachable services in the environment where each component runs. A localhost URL is meaningful only when the caller and destination share the same machine or network namespace. In a deployed backend, `localhost` refers to that backend container or process, not the separately deployed AI service.

The repository's historical QA report recorded an HTTP 500 from the AI prediction endpoint, and the active changelog later records another production smoke run with the same class of failure. The AI UI even presents a “server waking up” hint when the backend message contains “AI service failed.” That hint is a user-experience response to a possible cold start, not proof that every failure is a cold start. A bad URL, unavailable service, incompatible response, or runtime exception can produce the same broad symptom.

## Frontend request state is concurrency control

The report-list UI has had specific reliability work: request cancellation with `AbortController`, request identifiers so only the latest response updates state, and a debounce for filter-driven fetches. This is more than visual polish. If a user changes filters rapidly, an earlier request can finish after a newer request and overwrite the correct result. “Latest response wins” is an explicit client-side consistency rule.

The same principle should be applied carefully elsewhere: loading states must not permit accidental duplicate submissions, errors should give a useful recovery path, and a stale request should not overwrite current state. Cancellation is not guaranteed to stop work already performed by the server, so it should not be mistaken for backend idempotency.

## Uploads need content validation, not just a filename

The report workflow accepts evidence, and other parts of the product handle visual content. The changelog records replacing user-controlled upload names with generated UUID-based names, extension allowlists, memory-backed validation, MIME checks, magic-byte inspection with `file-type`, a 25 MB cap, and explicit persistence writes. These controls address different risks: generated names reduce path manipulation, size limits constrain resource consumption, and magic-byte checks make it harder to disguise one file type as another.

Validation still needs to be connected to storage lifecycle behavior. A successful write followed by a failed database operation can leave an orphaned file; a failed delete can leave data behind. The report controller attempts cleanup on failure and logs cleanup errors, but the documentation pass did not run filesystem failure tests or audit the deployed storage directory's access policy.

## Deployment order and operational checks

The deployment guide's sequence is backend, AI service, frontend configuration, then frontend deployment. It recommends checking health/version/uptime, verifying the backend can reach AI, testing authenticated report listing, confirming rate limits, and reviewing logs for secret exposure. These checks cover service connectivity and critical paths, but they should be supplemented with an end-to-end scenario that exercises the actual user journey.

The root README describes a dedicated TrustScan worker as planned. The reviewed system design is explicit that current TrustScan execution is controller-driven during status polling. Redis-backed queueing, retries, worker concurrency, and crash recovery must therefore remain future architecture until code and deployment prove otherwise.

## The release question

A green unit test suite cannot establish that Vercel points to the intended API, that Render has the required secrets, that the AI service can be reached, or that a browser can export a long report as a readable PDF. Conversely, one failed live smoke request does not identify the exact root cause by itself. The correct release record distinguishes automated tests, manual checks, live environment observations, and unresolved failures.

**Sources:** [System design](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/engineering/SYSTEM_DESIGN.md), [deployment guide](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/ops/DEPLOYMENT.md), [testing strategy](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/ops/TESTING.md), [AI client](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/services/aiService.js), [AI UI](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/client/src/pages/ai/ScamDetector.jsx), [upload middleware](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/middlewares/uploadMiddleware.js), [historical QA report](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/archive/qa-report.md).