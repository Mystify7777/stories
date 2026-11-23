# Chapter 11: Rewards Are Business Logic, Not Decoration

CyberShield includes XP, levels, streaks, coins, and badges. These features are visible engagement mechanics, but they also create an integrity problem: if a client can decide that an action was successful, it can decide that it deserves a reward. The backend therefore needs to own the rules that turn actions into progression.

## A shared progression model

The user schema stores XP, level, streak, coins, daily coin state, and badges. The gamification utility defines XP values for actions such as submitting a report, creating an article, using the AI detector, posting in the forum, creating or voting on memes, logging in, and answering a game question. Level calculation is deterministic: every 100 XP advances the calculated level, starting at level one. Badge checks award named milestones when the user's accumulated XP or streak crosses configured thresholds.

This centralization gives the product one place to express progression rules. But the implementation is not a separate event ledger: the XP helper loads the user, mutates the XP and level fields, checks badges, and saves the user. The changelog records later atomic updates for coin and XP progression, so the current source should be read carefully before claiming every reward path is fully atomic. A load-modify-save sequence can lose updates when concurrent actions target the same user unless concurrency is otherwise controlled.

## The phishing game as a security example

The game flow demonstrates why server authority matters. The public question endpoint returns questions, and the reward endpoint accepts a question ID and answer ID. The server validates the answer against its own question bank, rejects incorrect answers without granting a reward, checks a cooldown, records the last-played time, and then awards XP and attempts to award coins.

The changelog specifically records a change from trusting a client-provided `correct` boolean to validating the answer server-side. That closes a straightforward abuse path: a modified client can no longer merely assert that an answer was correct. The cooldown adds a basic rate boundary between successful attempts.

There are still distinct failure semantics. XP is awarded before the coin update, and a coin-economy error is caught so the XP can remain. That may be a deliberate resilience choice, but it means the reward is not one atomic transaction across both balances. A robust test should cover the partial-failure case and document whether the expected outcome is “XP only” or a complete rollback.

## Side effects across the product

Rewards are attached to actions in several controllers. Report and article submission, AI analysis, and community actions can increment engagement state. This creates a cross-cutting concern: retries, duplicate requests, and failures must not accidentally grant the same reward multiple times. Atomic increments help prevent lost updates, but they do not automatically provide idempotency. An endpoint needs a durable way to recognize a repeated logical action if duplicates must not earn duplicate rewards.

## Why it belongs in a security book

Gamification can improve engagement with awareness material, but reward state is also valuable user data. If XP or coins can be forged, leaderboard-like signals lose meaning and attackers can farm rewards. Server-side validation, cooldowns, bounded request rates, and careful update semantics protect the integrity of that system.

**Evidence limits:** The source shows the current controller/helper structure and the changelog's recorded hardening. The documentation pass did not run concurrent reward tests or verify economy behavior under database failures.

**Sources:** [Game controller](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/controllers/gameController.js), [gamification utility](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/utils/gamification.js), [economy utility](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/utils/economy.js), [user model](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/models/User.js), [active changelog](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/CHANGELOG.md).