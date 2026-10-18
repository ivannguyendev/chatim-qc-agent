# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

`chatim-qc-agent` is the QC automation repo: **Playwright + TypeScript** API and E2E suites, plus the AI pipeline that turns requirements into those suites. Inputs are PRDs, feature/module/API docs, screen captures and videos.

Input material is **transient**: it is handed over ad hoc and can be deleted at any time. Never link to an input file from code, docs or `INDEXES.csv`. Copy what must survive into `docs/` (requirements, checklists, test cases) or `src/constants/` (exact UI copy).

Quality goals:
- Give devs feedback within minutes of a PR.
- Zero critical regressions in production.
- Keep the revenue flows green: Auth, Onboarding, Checkout, Payment.
- Cut about 80% of manual regression work.
- Act as a quality gate that makes auto-deploy safe.

These goals decide what gets automated and in which layer.

**Status:** not scaffolded yet (no `package.json`). The layout and commands below are the target contract. Create them exactly as described when scaffolding.

## Commands (scripts the scaffold must provide)

The config lives in `config/playwright.config.ts`, so every raw CLI call needs `-c config/playwright.config.ts`. The npm scripts below wrap that.

```bash
npm ci && npx playwright install --with-deps
npm test                         # full suite (TEST_ENV=staging default)
npm run test:smoke               # tests/smoke, PR gate, budget < 5 min
npm run test:regression          # tests/regression
npm run test:api                 # tests/api (no browser)
npm run test:unit                # Vitest, pure helpers in src/utils
npm run typecheck                # tsc --noEmit
npm run lint
npx playwright test -c config/playwright.config.ts tests/regression/auth/login-scenarios.spec.ts  # one file
npx playwright test -c config/playwright.config.ts -g "TC-LOGIN-007"                               # one test
npx playwright test -c config/playwright.config.ts --grep "@p1" --project=chromium                 # tag/browser
npx playwright test -c config/playwright.config.ts <file> --repeat-each=10                         # flake proof
npx playwright show-report | npx playwright show-trace <trace.zip>
```

## Architecture

```
config/       env.staging.ts, env.production.ts (selected by TEST_ENV), playwright.config.ts
src/api/      API clients on Playwright APIRequestContext: login → token/storageState, seed/clean data
src/pages/    POM per module (auth/, transaction/…), all extend base.page.ts
src/components/ reusable UI parts (data-table, confirm-dialog), composed INTO pages
src/fixtures/ base.fixture.ts (artifacts, console capture), auth.fixture.ts (storageState per role)
src/constants/ endpoints, routes, regex, error-messages (verbatim UI copy from the spec)
src/utils/    data-generator (faker), db-helper (verify/cleanup only), date-time
tests/smoke/  critical path only, tagged @smoke
tests/regression/<module>/  detailed scenarios
tests/api/    integration/contract tests, no browser
test-data/    static payloads (mock-payloads/) and upload files (sample-files/)
.github/workflows/ test-pr-smoke.yml (PR), test-nightly-full.yml (full + browser matrix, sharded)
docs/         FINALIZED docs only: docs/<module>/<feature>/{checklist,testcases}.md
INDEXES.csv   codebase & context search index for agents (see below)
```

Layer rules. These are the reasons the files are split this way:
- **Specs hold flow and assertions only.** Pages and components expose locators and actions and never call `expect`. When the UI changes, the fix is one selector in one page or component file.
- **API-first setup.** Login, data seeding and teardown go through `src/api` clients, never through UI clicks. The UI test opens the target page directly, already authenticated. Use `APIRequestContext` rather than axios, so API calls show up in the trace.
- **Exact strings come from `src/constants/error-messages.ts`.** They are copied verbatim from the spec. Never inline or paraphrase toast and error text in specs.
- **db-helper is a last resort,** used only for verification or cleanup that no API exposes.
- **Production is read-only:** smoke only, with no seeding, teardown or db-helper, unless the user explicitly asks.
- Test locators, sync, storageState, seeding, isolation, artifacts and CI sharding follow `.claude/skills/playwright/SKILL.md` and its playbooks. Read the matching playbook before writing that kind of code; don't restate it here.

Test data: build names as `qa_auto_<workerIndex>_<randomUUID()>`, not `Date.now()`, which collides across parallel workers. The `qa_auto_` prefix lets leftover data be found and purged. Every test arranges and tears down its own data, and the suite must pass with 10 workers in any order.

## Pyramid in this repo

The app's 70% unit layer lives in the app repo, not here. This repo owns the **API layer (20%)** and **E2E layer (10%)**, plus Vitest unit tests for its own helpers. When you map a test case, choose the cheapest layer that can prove it:
- **`tests/api`:** business rules, calculations, limits, auth and permission, webhooks. Examples from the PRD: SePay 401 on a missing API key, the credit deduction formula, the 10-accounts-per-IP limit.
- **`tests/regression` (E2E):** behaviour only visible in the UI, such as validation messages, state transitions and plan gating.
- **`tests/smoke`:** `[1]` happy paths of the critical flows.

## Workflows: requirement → spec pipeline

> **Nguyên tắc cốt lõi:** Code generation BẮT BUỘC phải dựa trên `checklist.md` và `testcases.md`, trong đó `testcases.md` BẮT BUỘC phải sinh từ `checklist.md`. Agent tuyệt đối KHÔNG tự ý lên plan rồi nhảy vào viết code test tự do khi chưa có 2 tài liệu nền tảng này.

1. **Phase 1: AI Analyst (Checklist).**
   - Phân tích tài liệu yêu cầu (PRD, API doc, captures/videos).
   - Xuất checklist ra `docs/<module>/<feature>/checklist.md` theo phân cấp `#`, `##` với immutable ID `CHK-<FEATURE>-<NNN>`, độ ưu tiên `[P1..P4]` và test layer `[@api | @e2e | @ui]`.
   - *Ràng buộc:* Bắt buộc phải có file checklist này trước khi chuyển sang bước viết test case.

2. **Phase 2: AI Test Generator (Test Cases từ Checklist).**
   - Đọc `docs/<module>/<feature>/checklist.md` và mở rộng từng check item thành test case chi tiết trong `docs/<module>/<feature>/testcases.md` dưới dạng Structured Markdown Blocks (với Actors/Roles, Pre-conditions API Seeding, Test Data, Steps Action/Expected với verbatim UI error constants, và Post-conditions API Teardown).
   - Duy trì ánh xạ 1-1 chính xác với checklist (`CHK-<FEATURE>-<NNN>` → `TC-<FEATURE>-<NNN>`).
   - *Ràng buộc:* Toàn bộ test case phải sinh từ checklist, không tự ý bịa thêm test case ngoài checklist.

3. **Phase 3: Automation Code Gen (Sinh mã từ test case).**
   - BẮT BUỘC phải có đầy đủ `docs/.../checklist.md` và `docs/.../testcases.md` trước khi viết code.
   - Chỉ viết code (`src/` và `tests/`) bám sát đúng các test case đã có trong `testcases.md`:
     - `#` category → `test.describe`
     - `##` subcategory → nested `describe`
     - item → một `test('TC-<FEATURE>-<NNN> <title>', { tag: ['@p<priority>', ...] })`
     - steps → `test.step`
   - Phân tầng theo Test Pyramid (`tests/api/`, `tests/regression/`, `tests/smoke/`).
   - Tái sử dụng hoặc tạo mới POM (`src/pages/`), API client (`src/api/`), components (`src/components/`).

4. **Phase 4: Verify.**
   - Chạy kiểm thử với lệnh hẹp nhất (narrowest command). Báo cáo rõ command, exit code, pass/fail/flaky counts và trace path nếu có failure. Tuyệt đối không kết luận pass nếu chưa chạy test thực tế.

5. **Phase 5: Maintain Context Index.**
   - Cập nhật `INDEXES.csv` cho tất cả các file code, test, doc vừa tạo hoặc chỉnh sửa.

## Checklist & test case format

### Phase 1 Checklist

Format:

```markdown
F: <Feature name>
# Category
## Subcategory
- [P1][@e2e] CHK-<FEATURE>-001: check item description
- [P2][@api] CHK-<FEATURE>-002: check item description
```

Rules:
- **Immutable ID:** `CHK-<FEATURE>-<NNN>` generated immediately in Phase 1.
- **Priority:** `[P1]` (Critical), `[P2]` (High), `[P3]` (Medium), `[P4]` (Low).
- **Test Layer:** `[@api]`, `[@e2e]`, `[@ui]`, `[@unit]`.
- **"What, not How" principle:** Focus on business verification, never inline full tooltip strings or micro-steps into checklist.
- **Atomic action flows:** 1 check item = 1 scenario verifying 1 observable business outcome.
- **Exclude app header & footer** unless requested.
- **Ordering:** Order items by user flow, start to finish. Within each step, list happy paths first, then error cases, then edge cases.
- **Group checks by UI region:** For example, write one item for "all elements of region X display correctly", not one item per element.
- **Coverage & Depth:** Test types to cover per run: UI, Security, Performance, Accessibility, Data validation. Depth level: High (edge cases and rare scenarios, default), Medium (common cases), or Low (essential cases only).

### Phase 2 Test Case

Format (Structured Markdown Blocks):

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

Rules:
- **1-to-1 mapping with `CHK-<FEATURE>-<NNN>`:** Each checklist item corresponds directly to one test case.
- **Title & Description unified:** Merged cleanly into `<Scenario Title>`.
- **Dedicated Actors / Roles:** Mapping directly to `src/fixtures/auth.fixture.ts` `storageState`. Support multi-actor concurrency for real-time chat.
- **API Seeding in Pre-conditions:** State initialized via API using `qa_auto_<workerIndex>_<uuid>` isolation prefix.
- **Mandatory API Teardown in Post-conditions:** API cleanup call to prevent data pollution across 10 workers.
- **Verifiable Assertions:** Use verbatim UI copy from `src/constants/error-messages.ts` (`UI_MESSAGES.*`) and standardized DOM states (`Visible`, `Disabled`, or exact text).
- **NEVER use CSV or Markdown tables** for detailed test cases (preserve multi-line steps and git diff integrity).

## INDEXES.csv

`INDEXES.csv` is the codebase and context search index designed to help AI Agents find relevant files and complete context quickly and accurately without scanning the whole repository.

One row per file (source code, test spec, doc, config). Update it whenever creating, moving, or modifying files.

Columns:
`path,kind,purpose,key_symbols,used_by,tests,decisions`

| Column | Description / Allowed values |
|---|---|
| `path` | Relative path to the file (e.g. `src/pages/auth/login.page.ts`, `docs/auth/login/testcases.md`). |
| `kind` | File kind: `page` \| `component` \| `api` \| `fixture` \| `constant` \| `util` \| `spec` \| `doc` \| `config`. |
| `purpose` | Concise 1-line description of the file's domain and responsibility. |
| `key_symbols` | Exported classes, functions, interfaces, locators, constants (comma-separated, quote if commas). |
| `used_by` | Files or specs that import or invoke this file. |
| `tests` | Test specs covering or exercising this file. |
| `decisions` | Key architectural decisions, conventions, ADRs, or gotchas associated with this file. |

## Local-only material (gitignored on purpose; do not "fix" `.gitignore`)

- **`.claude/`, `.agent/`:** skills and plans. `.agent/skills` mirrors `.claude/skills`, so edit both.

## Domain gotchas

- Keep UI labels, button names and specialized terms in their original language, even inside Vietnamese test cases.
- When the LLM expands only part of a checklist, it echoes the parent `#` and `##` headers even though they weren't in the input. Strip the extra headers in code; prompt wording does not prevent it.
- Test data longer than 256 characters is truncated to about 30 characters plus `... (very_long_data)` in generated test cases. Real long payloads for automation go in `test-data/`.
