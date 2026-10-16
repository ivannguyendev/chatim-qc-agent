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
INDEXES.csv   traceability index (see below)
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

1. **Phase 1, AI Analyst.** Turn the input into a structured checklist (format in the next section). Use the `ai-multimodal` skill for captures and videos. Add the rows to `INDEXES.csv` with `status=checklist`.
2. **Phase 2, AI Test Generator.** Expand each item into a full test case (format in the next section). Set `status=designed`.
3. **Phase 3, Automation Code Gen.** Map test cases to specs:
   - `#` category → `test.describe`
   - `##` subcategory → nested `describe`
   - item → one `test('TC-<FEATURE>-<NNN> <title>', { tag: ['@p<priority>', ...] })`
   - steps → `test.step`

   Choose the layer using the pyramid rules above. Fill in `spec_file` and `layer` and set `status=automated`.
4. **Plan** non-trivial work in `<agent dir>/plans/YYMMDD-<feature>.md`: `.claude/plans/` for Claude, `.agent/plans/` for other agents (date from `date +%y%m%d`). Both folders are gitignored. Never put plans in `docs/`.
5. **Verify** with the narrowest command and report the command, exit code, pass/fail/flaky counts and the trace path for any failure. Never claim a pass without a run.
6. **Promote** the finalized checklist and test cases into `docs/<module>/<feature>/`.

## Checklist & test case format

**Phase 1 checklist:**

```
F: <Feature name>
# Category
## Subcategory
[1] check item
```

Rules:
- Order items by user flow, start to finish. Within each step, list happy paths first, then error cases, then edge cases.
- Group checks by UI region. For example, write one item for "all elements of region X display correctly", not one item per element.
- Merge the checks of a single action flow into one item. For example, "clicking 'Disconnect account' opens the confirmation popup" is one item, not two.
- Leave the app's header and footer out unless asked.

The test types to cover are chosen per run:
- UI
- Security
- Performance
- Accessibility
- Data validation

The depth level is also chosen per run: High (edge cases and rare scenarios, the default), Medium (common cases) or Low (essential cases only).

**Phase 2 numbering and expansion:**
- Categories are numbered `#1`, `#2`.
- Subcategories are numbered `##1.1`, `##1.2`.
- Items are numbered sequentially across the whole feature (`1.`, `2.`, …) regardless of subcategory. This number becomes the `NNN` in `TC-<FEATURE>-<NNN>`.
- The text after each item number stays exactly as written: never reword or translate it.

Each test case has this shape:

```
#1 Category
##1.1 Subcategory
1. [1] check item
Title: …
Pre-condition: …
Description: …
Steps:
Step: 1
Action: …
Expected: …
Data: … | N/A
```

## INDEXES.csv

One row per test case. Update it in the same change as the checklist, doc or spec it describes.

| Column | Values / rule |
|---|---|
| `tc_id` | `TC-<FEATURE>-<NNN>`. `NNN` is the sequential checklist item number. Must be unique. |
| `module`, `feature`, `category`, `subcategory` | Taken from the checklist hierarchy. |
| `priority` | `1`–`4` (1 Critical, 2 High, 3 Medium, 4 Low). |
| `check_item` | Text kept exactly as written. Quote it if it contains commas or quotes. |
| `layer` | `api` \| `e2e` \| `unit` |
| `suite` | `smoke` \| `regression` |
| `source_ref` | Stable origin of the requirement: a `docs/` path with an anchor, or an external ticket or URL. Never a transient input file. |
| `spec_file` | Path to the spec, empty until automated. |
| `status` | `checklist` → `designed` → `automated`, or `blocked` / `deprecated` |
| `updated` | `YYYY-MM-DD` |

## Local-only material (gitignored on purpose; do not "fix" `.gitignore`)

- **`.claude/`, `.agent/`:** skills and plans. `.agent/skills` mirrors `.claude/skills`, so edit both.

## Domain gotchas

- Keep UI labels, button names and specialized terms in their original language, even inside Vietnamese test cases.
- When the LLM expands only part of a checklist, it echoes the parent `#` and `##` headers even though they weren't in the input. Strip the extra headers in code; prompt wording does not prevent it.
- Test data longer than 256 characters is truncated to about 30 characters plus `... (very_long_data)` in generated test cases. Real long payloads for automation go in `test-data/`.
