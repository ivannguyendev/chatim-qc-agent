# CLAUDE.md

Guidance for Claude Code and AI Agents working in `chatim-qc-agent`.

## Project

`chatim-qc-agent` is the QC automation repo: **Playwright + TypeScript** API and E2E suites for real-time chat applications.
Input material (PRDs, captures, API docs) is **transient** and can be deleted anytime. Never link to transient input files from any tracked file (code, docs, CLAUDE.md, README, `INDEXES.csv`, `.memory/`). Copy permanent requirements into `docs/` and exact UI copy into `src/constants/`.

Quality goals:

- Fast PR feedback (< 5 min smoke gate).
- Zero critical regressions on core flows: Auth, Real-time Messaging (WebSocket), Channels/Groups, Attachments, Permissions.
- Cut ~80% of manual regression work via automated quality gates.

**Status:** Not scaffolded yet (no `package.json`). Create layout and files exactly as specified when scaffolding.

## Code index

`INDEXES.csv` at the repo root is the context-search entry point (columns `path,kind,purpose,key_symbols,used_by,tests`). Read it before searching the tree.

- **Row:** one file, folder or config, with the contexts that can be retrieved from it (listed in `purpose`/`key_symbols`). No line numbers.
- **Scope:** only official files, folders, configs and docs that are confirmed correct and useful as retrieval context. Never index transient inputs (`.local/`), plans or drafts.
- **Update:** for every new or modified file, check whether it is in scope or changes an existing row; if so, add or update the row in the same change.

## Repo Memory

`.memory/` is the shared, tracked memory for all agents (context, decisions and the *why* behind rules). Read `.memory/MEMORY.md` at the start of every session.

- Write new memories there (one fact per file, frontmatter `name/description/metadata.type`, index line in `.memory/MEMORY.md`); update an existing file instead of duplicating.
- Rules belong in this file; `.memory/` explains them and must not copy CLAUDE.md or TESTING.md content.
- Never store secrets or links to transient input files (`.local/`).

## Architecture & Commands

Config lives at `config/playwright.config.ts` (all raw CLI calls require `-c config/playwright.config.ts`).

```
config/       env.staging.ts, env.production.ts (selected by TEST_ENV), playwright.config.ts
src/api/      Playwright APIRequestContext clients: login → token/storageState, seed/clean data
src/pages/    POM per module (auth/, chat/…), extend base.page.ts; NO assertions in POM
src/components/ Reusable UI parts (dialogs, message-input, user-list), composed into pages
src/fixtures/ base.fixture.ts (artifacts), auth.fixture.ts (storageState per role: user, leader, admin)
src/constants/ endpoints, routes, regex, error-messages.ts (verbatim UI copy constants UI_MESSAGES.*)
src/utils/    data-generator (faker), date-time, db-helper (last resort only)
tests/smoke/  Critical path only (@smoke, budget < 5 min)
tests/regression/<module>/ Detailed E2E scenarios
tests/api/    Fast contract/integration tests without browser (business rules, limits, status codes)
test-data/    mock-payloads/ and sample upload files (sample-files/)
docs/         FINALIZED docs only: docs/<module>/<feature>/{checklist,testcases}.md, docs/plans/*-summary.md
.memory/      Shared agent memory (context, decisions, rationale); index in .memory/MEMORY.md
TESTING.md    Test engineering standards, checklist & testcase format rules, Playwright mappings
INDEXES.csv   Codebase context search index: path,kind,purpose,key_symbols,used_by,tests
```

Key npm scripts:

```bash
npm test                         # full suite (TEST_ENV=staging default)
npm run test:smoke               # tests/smoke, PR gate, budget < 5 min
npm run test:regression          # tests/regression
npm run test:api                 # tests/api (no browser)
npm run test:unit                # Vitest, pure helpers in src/utils
npm run typecheck                # tsc --noEmit
npm run lint
```

## Testing Rules & Pyramid

- **Pyramid:** App repo owns unit tests (70%). This repo owns **API tests (20%)** and **E2E tests (10%)**. Choose the cheapest layer that can prove a behavior.
  - `tests/api/`: Business logic, permissions, payload validation, status codes, rate limits.
  - `tests/regression/`: UI workflows, state transitions, validation feedback, cross-component interactions.
  - `tests/smoke/`: Happy paths of critical user journeys.
- **API-First Setup:** Login, data seeding, and teardown MUST use `src/api/` clients, never UI clicks. UI tests open pages directly, already authenticated.
- **Multi-Actor Concurrency:** Chat scenarios involving real-time message exchange require multiple browser contexts (Sender vs Receiver) using `authFile('<role>')`.
- **Data Isolation:** Prefix all dynamic entities with `qa_auto_<workerIndex>_<randomUUID()>` (never `Date.now()`). Each test arranges and tears down its own data to ensure parallel safety across 10 workers.
- **Exact UI Copy:** Assertions must use exact constants from `src/constants/error-messages.ts`. Never paraphrase toast/error text.
- **Production is Read-Only:** Smoke tests only; no seeding, teardown, or db-helpers in production.

## Agent Workflow (Mandatory)

Applies to every non-trivial task (new feature or test suite, refactor, standards change). `<agent dir>` is `.claude` for Claude, `.agent` for other agents; `YYMMDD` comes from `date +%y%m%d`.

1. **Brainstorm:** read README, CLAUDE.md, `.memory/`, `INDEXES.csv` and the relevant docs. **Evidence first:** every requirement, checklist item and test case must trace to a source (PRD content copied into `docs/`, `checklist.md`, `testcases.md`, or an explicit user decision). Never invent behavior, UI copy, limits or endpoints. If the source is missing or unclear, stop and present the goal, a proposed component breakdown and the open questions; an incomplete breakdown will not be approved.
2. **Save decisions:** `<agent dir>/plans/YYMMDD-<feature>-design.md` with decisions, assumptions, rejected alternatives (with why) and open risks.
3. **Reviewer: phase split:** before any plan summary, a reviewer assesses the scope and discusses with the user whether to split it into milestones/phases. Each phase must be deliverable and verifiable on its own so later changes stay isolated.
4. **Plan summary (current phase only):** `docs/plans/YYMMDD-<feature>-p<N>-summary.md` with goal, scope/out-of-scope, deliverables, acceptance criteria, evidence sources and risks. The user approves it.
5. **Detailed plan:** `<agent dir>/plans/YYMMDD-<feature>-p<N>.md`, written only after the phase summary is approved.
6. **Review:** a reviewer checks the detailed plan against the summary and evidence; resolve all Critical/Important findings before coding.
7. **Code:** follow the detailed plan; report every deviation.
8. **Test:** run the narrowest command; report exit code and pass/fail/flaky; update the phase summary to the actual build; update `INDEXES.csv` per "Code index" (Phase 5).
9. **Close-out:**
   - Rule check: look for new rules, conventions or decisions the user confirmed during the task; update CLAUDE.md (rules) and copy confirmed decisions from the design log into `.memory/` (rationale), since `<agent dir>/plans/` is gitignored; state what changed or "no new rules".
   - Progress: state the phase just finished, the next phase, and all remaining phases.

For test work, Phases 1–2 below (`checklist.md`, `testcases.md`) are produced and approved as deliverables of steps 4–6 and become the evidence gate; Phases 3–5 run in steps 7–8.

## 5-Phase Workflow Pipeline

> **BẮT BUỘC:** Không viết code test khi chưa có `checklist.md` và `testcases.md`. Code generation phải bám sát 100% test case đã thiết kế. Chi tiết xem tại [TESTING.md](TESTING.md).

1. **Phase 1: AI Analyst (Checklist)** $\rightarrow$ `docs/<module>/<feature>/checklist.md`
   - Phân cấp `#`, `##` với immutable ID `CHK-<FEATURE>-<NNN>`, tag `[P1..P4]` và `[@api | @e2e | @ui]`.
   - Nguyên tắc: "What, not How" & Action-Feedback loop (không tách vụn vặt UI element, không ghép 2 luồng độc lập).
2. **Phase 2: AI Test Generator (Test Cases)** $\rightarrow$ `docs/<module>/<feature>/testcases.md`
   - Mở rộng checklist 1-1 thành Structured Markdown Blocks (`### TC-...`), bổ sung Actors/Roles, API Seeding, API Teardown, Verifiable Assertions.
   - **Tuyệt đối không dùng bảng CSV / Markdown table cho test case chi tiết.**
3. **Phase 3: Automation Code Gen** $\rightarrow$ `tests/`
   - Map 1-1: `#` $\rightarrow$ `test.describe`, `TC-...` $\rightarrow$ `test()`, steps $\rightarrow$ `test.step()`.
4. **Phase 4: Verify** $\rightarrow$ Chạy lệnh test hẹp nhất, báo cáo exit code, pass/fail/flaky và trace.
5. **Phase 5: Maintain Context Index** $\rightarrow$ Cập nhật `INDEXES.csv` theo rule tại mục "Code index".

## Formats Quick Reference

### Phase 1 Checklist

```markdown
F: <Feature name>

# Category

## Subcategory

- [P1][@e2e] CHK-<FEATURE>-001: check item description
- [P2][@api] CHK-<FEATURE>-002: check item description
```

### Phase 2 Test Case (Structured Markdown Block)

```markdown
### TC-<FEATURE>-<NNN>: <Scenario Title>

- **Trace:** CHK-<FEATURE>-<NNN> | **Priority:** [P1] | **Layer:** @e2e | **Suite:** smoke | **Role:** user
- **Actors:**
  - Actor 1 (Primary): Role `user` (storageState: `authFile('user')`)
  - Actor 2 (Receiver, optional): Role `leader` (storageState: `authFile('leader')`)
- **Pre-conditions (API Seeding):** State initialized via API using `qa_auto_<workerIndex>_<uuid>`
- **Test Data:** Payloads or exact constants from `src/constants/`
- **Steps:**
  1. **Action:** Navigate to ...
     **Expected:** Verifiable DOM state (`Visible`, `Disabled`, or exact text)
  2. **Action:** ...
     **Expected:** ...
- **Post-conditions (API Teardown):** API cleanup call
```

## Definition of Done for Merging to Main

1. **Plan & Summary Completion:** All tasks in the phase plan (`<agent dir>/plans/YYMMDD-<feature>-p<N>.md`) are complete, and `docs/plans/YYMMDD-<feature>-p<N>-summary.md` reflects the actual build; any steps deviating from "Expected" have been reported and resolved.
2. **All Green:** Both `npm run lint` and all test suites (`typecheck`, `test:unit`, `test`) pass cleanly.
3. **No Critical / Important Findings:** Zero unresolved Critical or Important findings; all Minor findings are formally documented in the plan or docs.
4. **Go-Live Readiness in PR:** PR description clearly states the readiness level and any remaining items required before production go-live.

## Domain Gotchas

- Keep UI labels, button names, and specialized terms in their original language.
- Decision logs and detailed plans belong in `.claude/plans/` or `.agent/plans/` (gitignored like `.local/`; never un-ignore them). Never put detailed plans in `docs/`; phase plan summaries belong in `docs/plans/YYMMDD-<feature>-p<N>-summary.md`.
