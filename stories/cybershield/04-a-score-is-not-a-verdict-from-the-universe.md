# Chapter 4: A Score Is Not a Verdict from the Universe

TrustScan uses a score from 0 to 100, with higher values intended to indicate lower risk. Its documented factor families cover transport security, infrastructure/domain signals, reputation, and header hygiene. The score is accompanied by a verdict band, confidence level, factor-level evidence, and a human-readable summary.

The current scoring implementation starts from a base score, sums factor impacts, clamps the result to 0–100, and applies a confidence ceiling. High confidence can reach 100, Medium is capped at 89, and Low is capped at 75. Verdict bands are then derived from the resulting score.

This ceiling is a guard against false precision: incomplete evidence should not silently become the most reassuring possible score. It does not make the underlying signals complete or the score a guarantee.

## Confidence depends on coverage and failures

The confidence logic examines the available signal sources and whether they report a reason other than success. Four healthy sources yield High. Three or more sources with at most one degraded source can yield Medium. Otherwise confidence is Low. The scoring module contains its own confidence derivation and cap logic, while a separate confidence service exposes related behavior to the controller. That duplicated policy deserves regression coverage whenever either implementation changes.

## Human-readable explanations

The summary builder creates a sentence from TLS status, missing headers, domain age/resolution, and reputation availability or findings. This is more interpretable than returning only a number, but the wording still inherits the limitations of the input signals. “No known public listings” is not the same claim as “this site is safe,” and “service unavailable” is not the same as “clean.”

## What the score does not establish

A weighted score is a decision aid, not a proof of maliciousness or legitimacy. It depends on external signal quality, reachability, registration data availability, and policy choices encoded in the weights and caps. The policy and product documents explicitly call for confidence framing and transparent boundaries.

**Sources:** [Scoring model](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/security/TRUSTSCAN_SCORING.md), [scoring implementation](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/services/trustscan/scoringService.js), [confidence service](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/services/trustscan/confidenceService.js), [summary builder](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/services/trustscan/summaryService.js), [factor builders](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/services/trustscan/factorBuilders.js).
