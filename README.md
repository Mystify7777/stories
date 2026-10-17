# GitHub Stories

A curated, evidence-based record of the projects, engineering decisions, and contributions behind my GitHub work.

## Documentation principles

- Document public repositories only. Private repositories are excluded from public documentation.
- Separate verified implementation from plans, aspirations, and historical context.
- Credit collaborative work accurately; distinguish original project ownership from contributions to forks and cohort repositories.
- Treat repository READMEs as evidence, not unquestionable truth. Verify important claims against code, tests, history, and project documentation.
- Do not publish credentials, tokens, personal secrets, or other sensitive data.
- Do not modify source repositories as part of this documentation workflow.

## Stories

Substantial projects are documented as chapter-based technical books, with a contents page and one Markdown file per chapter. The narrative covers the product problem, architecture, implementation, trade-offs, chronology, tests, and limitations.

- [DevLens](stories/devlens.md) — short-form engineering story; book expansion remains future work.
- [HaloTask Pro](stories/halotask-pro/README.md) — draft book, pending review and verification.
- [CyberShield](stories/cybershield/README.md) — draft book, pending review and verification.

Drafts are not publication claims. Each book identifies the source revision reviewed and separates repository claims from checks actually performed.

## Workflow

1. Inventory public repositories and classify them.
2. Inspect repository documentation, source structure, tests, history, and contribution evidence.
3. Draft documentation on a dedicated branch.
4. Review the diff for accuracy, privacy, attribution, and unsupported claims.
5. Open a pull request for review before merging.

The initial commit establishes this repository. All subsequent documentation changes must use a dedicated branch and pull request.
