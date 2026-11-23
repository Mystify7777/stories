# GitHub Stories

A curated, evidence-based library of technical books about the projects, engineering decisions, and contributions behind my GitHub work.

## Documentation principles

- Document public repositories only. Private repositories are excluded from public documentation.
- Substantial projects receive a book: one folder per project, one Markdown file per chapter, and a project README that serves as the book's contents page.
- Write narrative technical nonfiction, not a feature inventory or compressed audit report. Explain the problem, decisions, implementation, failures, trade-offs, tests, and evolution.
- Separate source-verified behavior from README claims, plans, historical context, and unverified production behavior.
- Credit collaborative work accurately; distinguish original project ownership from contributions to forks and cohort repositories.
- Do not publish credentials, tokens, personal secrets, or other sensitive data.
- Do not modify source repositories as part of this documentation workflow.

## The library

### Engineering books

- [HaloTask Pro: The Work Between Click and Consistency](stories/halotask-pro/README.md) — an offline-first productivity application, its sync queue, account isolation, task history, Growth Tree, reminders, and authentication.

### Project research

- [Public repository inventory](docs/repository-inventory.md) — a provisional discovery index, not a substitute for individual project audits.

DevLens has an earlier short-form story at [stories/devlens.md](stories/devlens.md). It is not yet in the new chapter-book format and should be expanded into a full book in a later pass. Mystify Observatory remains excluded from the active story lineup while it is under development.

## Book standard

Each book should have a coherent chapter sequence. Every chapter should develop a technical question, explain the relevant reasoning and implementation, cite evidence, and connect to the wider narrative. Chapters should be substantial and self-contained without repeating the same overview. Diagrams, code excerpts, tests, decisions, and links should be used where they clarify the story.

A book must not invent incidents or motivations to make the narrative dramatic. When evidence is missing, say so. Test files are not the same as executed test results, and deployed URLs are not proof of production health.

## Workflow

1. Inventory the public repository.
2. Inspect documentation, source, tests, configuration, and relevant history.
3. Design the chapter outline from evidence before writing.
4. Draft each chapter as a separate file inside the project folder.
5. Review the complete book for technical accuracy, privacy, attribution, chronology, transitions, and repetition.
6. Open a pull request from a dedicated branch for review before merging.

The initial commit established this repository. All subsequent documentation changes must use a dedicated branch and pull request.
