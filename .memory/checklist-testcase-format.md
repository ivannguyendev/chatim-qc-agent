---
name: checklist-testcase-format
description: Where the checklist/test case format lives, the 2026-10-18 format decisions with rejected alternatives, and the header-echo chunking pain point
metadata:
  type: reference
---

Canonical format: CLAUDE.md ("Formats Quick Reference") and TESTING.md (full standard, quality gates, Playwright mapping). The older `[1-4] item` / `#1`, `##1.1` numbering from the user's first prompts is superseded.

Decisions approved 2026-10-18 (problems they fix: missing layer tag made everything E2E; missing role broke `storageState`; missing teardown polluted data; loose markdown broke chunking):
- Immutable `CHK-` IDs from Phase 1; `[P1..P4]` and `[@api|@e2e|@ui]` tags required.
- Test cases add Actors/Roles (multi-actor for real-time chat), API seeding, API teardown, assertions against `UI_MESSAGES.*` constants; Title and Description merged.
- Rejected: detailed test cases in CSV/Excel/markdown tables (LLMs break quotes/commas on multi-line cells; git diff becomes unreadable).
- Rejected: raw Playwright code (`expect`, `toBeVisible()`) inside test cases (spec must stay readable by manual QC and stakeholders; use normalized states such as `Disabled`).

Known pain point the user could not solve by prompting: when only part of a checklist is expanded, the LLM echoes parent `#`/`##` headers. Strip them in code rather than relying on the prompt.

Related: [[test-architecture-decisions]].
