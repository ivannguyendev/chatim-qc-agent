# CLAUDE.md

Guidance for Claude Code and AI Agents working in `chatim-qc-agent`.

## Project

`chatim-qc-agent` is the QC automation repo: **Playwright + TypeScript** API and E2E suites for real-time chat applications.
Input material (PRDs, captures, API docs) is **transient** and can be deleted anytime. Never link to transient input files from code, docs, or `INDEXES.csv`. Copy permanent requirements into `docs/` and exact UI copy into `src/constants/`.

Quality goals:
- Fast PR feedback (< 5 min smoke gate).
- Zero critical regressions on core flows: Auth, Real-time Messaging (WebSocket), Channels/Groups, Attachments, Permissions.
- Cut ~80% of manual regression work via automated quality gates.

**Status:** Not scaffolded yet (no `package.json`). Create layout and files exactly as specified when scaffolding.

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
docs/         docs/<module>/<feature>/{checklist,testcases}.md, docs/standards/
INDEXES.csv   Codebase context search index: path,kind,purpose,key_symbols,used_by,tests,decisions
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

## 5-Phase Workflow Pipeline

> **BẮT BUỘC:** Không viết code test khi chưa có `checklist.md` và `testcases.md`. Code generation phải bám sát 100% test case đã thiết kế. Chi tiết xem tại `docs/standards/checklist-and-testcase-spec.md`.

1. **Phase 1: AI Analyst (Checklist)** $\rightarrow$ `docs/<module>/<feature>/checklist.md`
   - Phân cấp `#`, `##` với immutable ID `CHK-<FEATURE>-<NNN>`, tag `[P1..P4]` và `[@api | @e2e | @ui]`.
   - Nguyên tắc: "What, not How" & Action-Feedback loop (không tách vụn vặt UI element, không ghép 2 luồng độc lập).
2. **Phase 2: AI Test Generator (Test Cases)** $\rightarrow$ `docs/<module>/<feature>/testcases.md`
   - Mở rộng checklist 1-1 thành Structured Markdown Blocks (`### TC-...`), bổ sung Actors/Roles, API Seeding, API Teardown, Verifiable Assertions.
   - **Tuyệt đối không dùng bảng CSV / Markdown table cho test case chi tiết.**
3. **Phase 3: Automation Code Gen** $\rightarrow$ `tests/`
   - Map 1-1: `#` $\rightarrow$ `test.describe`, `TC-...` $\rightarrow$ `test()`, steps $\rightarrow$ `test.step()`.
4. **Phase 4: Verify** $\rightarrow$ Chạy lệnh test hẹp nhất, báo cáo exit code, pass/fail/flaky và trace.
5. **Phase 5: Maintain Context Index** $\rightarrow$ Cập nhật `INDEXES.csv` cho các file mới/chỉnh sửa.

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

## Domain Gotchas

- Keep UI labels, button names, and specialized terms in their original language.
- Local plans belong in `.claude/plans/` or `.agent/plans/` (both gitignored). Never put plans in `docs/`.
