---
name: test-architecture-decisions
description: Why the test architecture looks the way it does — pyramid split, API-first setup, POM without assertions, qa_auto data naming
metadata:
  type: project
---

The layout and rules themselves are canonical in CLAUDE.md ("Architecture & Commands", "Testing Rules & Pyramid"). This file keeps the reasons the user gave on 2026-10-17:

- Pyramid 70/20/10: unit tests live in the app repo; this repo owns API (`tests/api`) and E2E (`tests/smoke`, `tests/regression/<module>`), plus Vitest for its own helpers. Each behavior goes to the cheapest layer that proves it.
- API-first login/seed/teardown: UI setup is slow and flaky; the UI test opens the target page already authenticated.
- POM and component objects hold no `expect`, so specs own flow and assertions.
- Unique data `qa_auto_<workerIndex>_<uuid>`: the user's first draft used `Date.now()`, which collides across 10 parallel workers.

**Why:** fast PR feedback and zero flakiness under 10 parallel workers.
**How to apply:** follow CLAUDE.md when scaffolding or writing specs. Related: [[checklist-testcase-format]].
