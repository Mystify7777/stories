# Chapter 7: Security Controls, Tests, and What Remains Unproven

CyberShield's changelog records a substantial set of security and reliability changes: refresh-cookie authentication, short-lived access tokens, OTP hardening, error masking, upload validation, report encryption, URL validation, rate limits, database indexes, and server-side game-answer validation. These are meaningful engineering themes because they move trust decisions away from user-controlled input and into server-side checks.

A changelog is evidence of the project's recorded work, not by itself proof that every control is correct in the current deployment. Each security claim should be checked against the corresponding source and regression tests. This book does not independently certify CyberShield or perform a penetration test.

## Controls visible in the reviewed source

The Express app configures Helmet, CORS allowlisting, sanitization middleware, rate limiting, JSON parsing, cookie parsing, and modular routes. TrustScan has route-level validation and authentication middleware for job creation, history, and private status retrieval, while the public report endpoint is intentionally read-only. The report controller encrypts descriptions marked sensitive and has a public serializer that masks sensitive details.

The changelog also describes defensive upload handling: generated filenames, extension allowlists, magic-byte validation, and a hard upload-size limit. These should be evaluated together with cleanup behavior, storage permissions, and error handling rather than as isolated checklist items.

## Tests and evidence boundaries

The TrustScan route suite uses Vitest, Supertest, and mocks for the user/job/report models and external signal functions. This can verify route logic and expected response shapes under controlled inputs. It does not prove actual DNS, TLS, reputation-service behavior, Mongo persistence, production environment variables, or PDF rendering.

The README reports 55 backend tests passing, and a CI workflow exists for server tests. This review did not run tests or inspect a fresh workflow result, so the count remains a repository claim rather than a reproduced result.

The checked-in smoke script calls the configured backend base URL and checks guest/failure paths such as health, version, public feeds, and AI prediction. The testing guide separately requires manual user/admin scenarios and TrustScan scenarios, including malformed URLs, denied targets, status transitions, slow polling, public reports, and PDF export. A smoke suite is not a substitute for those flows.

## Verification queue before publication

- Run the backend test suite and record the exact command, count, skips, and failures.
- Run the smoke suite against an explicitly chosen target, without exposing credentials or sensitive data.
- Verify AI calls from the deployed backend reach the configured AI service.
- Exercise TrustScan from URL submission through persisted report, history, public sharing, and PDF export.
- Test denied targets and confirm no prohibited network access is attempted.
- Inspect PDF output for long reports, missing assets, page sizing, and readable content.
- Check whether current production deployment settings match the documented environment variables and origins.
- Reconcile implementation with changelog claims and confirm which reliability features remain planned.

**Sources:** [Security guide](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/security/SECURITY.md), [TrustScan tests](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/tests/routes/trustscan.routes.test.js), [testing strategy](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/ops/TESTING.md), [smoke script](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/scripts/qaSmoke.js), [changelog](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/CHANGELOG.md), [Express app](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/app.js).
