# Engineering Decisions and Evidence Timeline

This page records the main engineering directions visible in the reviewed public repository. It is not a complete commit-by-commit history. The implementation links are pinned to [source revision `740a612024491ecfb0df19f869a02be68d2f55b7`](https://github.com/Mystify7777/CyberShield/commit/740a612024491ecfb0df19f869a02be68d2f55b7); the repository's active changelog is the primary chronology source where commit-level attribution was not independently reconstructed.

## Product and architecture direction

- **Multiple trust workflows:** reporting/moderation, AI text classification, and TrustScan are part of one platform, but have different inputs, risk boundaries, and evidence requirements.
- **Separate AI runtime:** the Express API calls a FastAPI prediction service through `AI_SERVICE_URL`. This gives deployment independence but makes runtime configuration and connectivity part of the product's correctness.
- **Passive-only URL analysis:** the TrustScan policy restricts checks to passive signals and explicitly excludes intrusive scanning and exploit attempts.
- **Evidence-backed scoring:** factors, confidence caps, a verdict, and a generated summary accompany the score. The goal is to reduce false certainty, not eliminate uncertainty.
- **Job/report separation:** job state and completed report artifacts use separate persistence models and distinct routes for authenticated history and public read-only sharing.
- **Background worker remains future work:** the architecture documents describe a dedicated worker and Redis queue as planned. The reviewed controller advances execution in the status-polling completion path.
- **Client-side PDF export:** report DOM is captured into an image and embedded in a PDF. It is an implemented export path, but browser rendering needs end-to-end checks.

## Security and reliability themes recorded in the changelog

The active changelog records work on server-side game-answer validation, upload validation and generated filenames, URL validation and private-target rejection, refresh-cookie authentication, OTP hardening, generic 5xx errors, report encryption, rate limits, pagination caps, database indexes, and atomic reward updates. These entries guide where source and tests should be inspected; they are not a replacement for that inspection.

## Selected implementation references

- [Active changelog](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/CHANGELOG.md)
- [System design](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/engineering/SYSTEM_DESIGN.md)
- [TrustScan architecture](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/engineering/TRUSTSCAN_ARCHITECTURE.md)
- [TrustScan policy](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/security/TRUSTSCAN_POLICY.md)
- [TrustScan scoring implementation](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/services/trustscan/scoringService.js)
- [TrustScan route tests](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/tests/routes/trustscan.routes.test.js)
- [AI adapter](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/services/aiService.js)
- [PDF export helper](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/client/src/utils/trustscanPdf.js)

## Evidence gaps

The source documents call out retries, timeouts, crash recovery, and future queue/worker infrastructure. This book does not treat those as verified guarantees unless current code and tests demonstrate them. Production health, live AI integration, real external-signal behavior, end-to-end report completion, and PDF quality were not tested during this documentation pass.


## The wider platform is part of the engineering story

The reviewed source is not only a scanner. Its additional systems establish separate trust boundaries and deserve independent treatment:

- **Incident reporting:** structured taxonomy, anonymous and sensitive reports, encrypted sensitive descriptions, evidence upload, public-safe projections, moderation states, and non-transactional side effects.
- **Identity and sessions:** OTP verification and recovery flows, short-lived access tokens, refresh cookies and rotation, logout, role checks, and suspension state.
- **AI-assisted triage:** a separately deployed FastAPI keyword heuristic called through a configured backend URL. Its fixed confidence values and deployment dependency are part of the story, not marketing footnotes.
- **Community and awareness:** article approval, forum posts/replies, video submissions, meme voting, and admin moderation. These workflows need both content governance and ordinary application security.
- **Gamification:** XP, levels, streaks, coins, badges, server-side game-answer validation, cooldowns, and reward update semantics.
- **Administration and observability:** role hierarchy, report/user/content management, persistent counters, health/version/uptime endpoints, and client-error ingestion.
- **Cross-cutting operations:** upload validation, error masking, rate limits, database indexes, bounded pagination, environment validation, frontend request ordering, and independent service deployment.

These features should not be forced into one chapter called “miscellaneous.” They have different invariants and failure modes. The chapters explain each workflow and then connect them through shared concerns such as authorization, side effects, privacy, observability, and release verification.

The same evidence rule applies across the book: the active changelog records intended changes, but a claim about current behavior should be checked against the implementation at the pinned source revision. Historical smoke reports are historical; planned infrastructure remains planned; tests not executed during this documentation pass are not described as reproduced.
