# Checklist & Test Case Specification Guide

This guide establishes the canonical specification standards for quality engineering in the `chatim-qc-agent` repository. It defines how AI agents and human QC engineers must structure, author, and validate **Phase 1 Checklists** (`checklist.md`) and **Phase 2 Test Cases** (`testcases.md`), and how these documents map deterministically into Playwright test suites.

---

## 1. Core Philosophy & Quality Pipeline

### 1.1 The 5-Phase End-to-End Pipeline

Quality engineering in this repository operates on a strict, unidirectional pipeline:

```
+-------------------------------------------------------------------------------+
|                               5-PHASE PIPELINE                                 |
+-------------------------------------------------------------------------------+
|  Phase 1: AI Analyst (Checklist)                                              |
|  Inputs: PRD, API docs, UI captures, videos (transient)                       |
|  Output: docs/<module>/<feature>/checklist.md                                 |
|          -> Risk Coverage Map (WHAT)                                          |
+-------------------------------------------------------------------------------+
                                      |
                                      v
+-------------------------------------------------------------------------------+
|  Phase 2: AI Test Generator (Test Cases)                                      |
|  Inputs: docs/<module>/<feature>/checklist.md                                 |
|  Output: docs/<module>/<feature>/testcases.md                                 |
|          -> Deterministic Execution Spec (HOW)                                |
+-------------------------------------------------------------------------------+
                                      |
                                      v
+-------------------------------------------------------------------------------+
|  Phase 3: Automation Code Generation                                          |
|  Inputs: docs/<module>/<feature>/testcases.md                                 |
|  Outputs: tests/smoke/, tests/regression/, tests/api/, src/pages/, src/api/   |
|           -> 1-to-1 Playwright execution code                                 |
+-------------------------------------------------------------------------------+
                                      |
                                      v
+-------------------------------------------------------------------------------+
|  Phase 4: Verification & Triage                                               |
|  Action: Run narrowest test command, inspect traces, repeat for stability     |
|          -> Command output, pass/fail metrics, trace path                     |
+-------------------------------------------------------------------------------+
                                      |
                                      v
+-------------------------------------------------------------------------------+
|  Phase 5: Context Index Maintenance                                           |
|  Action: Record newly created/modified artifacts into INDEXES.csv             |
|          -> Fast agent context discovery and cross-module traceability        |
+-------------------------------------------------------------------------------+
```

> **Cardinal Rule:** Automation code generation **MUST** be derived from `checklist.md` and `testcases.md`. Test cases in `testcases.md` **MUST** be derived directly from `checklist.md`. Agents and engineers must **NEVER** write ad-hoc automation code without these foundational specification documents.

### 1.2 "What vs How": Risk Coverage Map vs Deterministic Execution Spec

The separation of concerns between Phase 1 and Phase 2 prevents cognitive overload and eliminates brittle, prematurely detailed specifications:

| Dimension | Phase 1: Checklist (`checklist.md`) | Phase 2: Test Cases (`testcases.md`) |
| :--- | :--- | :--- |
| **Primary Goal** | **WHAT** needs to be verified | **HOW** to execute verification deterministically |
| **Perspective** | Business risk coverage & requirement completeness | Automated execution spec & human test procedure |
| **Abstraction Level** | High level: observable business outcomes | Concrete level: steps, locators, payloads, assertions |
| **Pre-conditions** | Implicit or brief business state | Explicit API seeding scripts and token states |
| **Data Requirements** | Classes of input (valid PNG, oversized file) | Exact fixture paths, byte sizes, payload schemas |
| **Assertion Strategy** | Expected business result (message received) | Verifiable DOM states, HTTP codes, verbatim UI copy |
| **Teardown** | Not detailed | Explicit API cleanup calls to purge test artifacts |

### 1.3 Test Pyramid Alignment in this Repository

Per `CLAUDE.md`, the unit testing layer (70%) resides inside the application repository. This repository focuses strictly on the test layers it owns:

```
                      /\
                     /  \
                    / 10%\   E2E Layer (Playwright Browser)
                   /------\  tests/regression/, tests/smoke/
                  /  20%   \ API Layer (Headless Playwright APIRequestContext)
                 /----------\ tests/api/
                /    70%     \ Unit Layer (Lives in Application Codebase)
               / (Out of Scope\ Pure helpers only in src/utils via Vitest
              /________________\
```

- **API Layer (20% - `tests/api/`):**
  - Fastest, cheapest layer for proving functional logic.
  - Covers business calculations, credit deductions, permission matrix, rate limits, webhooks, payload schemas, and HTTP error codes.
  - Rule: If a requirement does not require browser rendering or client-side JavaScript execution, it belongs in `tests/api/`.
- **E2E Layer (10% - `tests/regression/` & `tests/smoke/`):**
  - High-confidence verification of critical user journeys.
  - Covers multi-actor synchronization, drag-and-drop file uploads, UI toasts/modals, client-side route guards, and visual feedback states.
  - Sub-suites:
    - `tests/smoke/`: Revenue-critical happy paths (`[P1]`, budget < 5 min for PR gate).
    - `tests/regression/`: Deep functional scenarios, edge cases, error recovery.

### 1.4 Why Structured Markdown Blocks (Not CSV or Markdown Tables)

Earlier test management practices relied on tabular representations (CSV or Markdown tables). In production test automation, tabular formats collapse under real-world requirements:

1. **Multi-line Steps & Payload Integrity:**
   Test steps require multi-line action descriptions, structured JSON payloads, and multiple assertions. Inside table cells, line breaks require clumsy `<br>` tags or escaped strings that break markdown formatters and LLM tokenizers.
2. **Git Diff Readability:**
   A single change inside a table row frequently triggers diff collisions across adjacent columns. Structured Markdown Blocks isolate every metadata attribute, step, and assertion to its own line, producing clean, mergeable git diffs.
3. **Zero Parser Breakage:**
   Pipes (`|`), quotes (`"`), and commas inside test data (e.g., regex patterns, SQL/JSON fixtures) frequently corrupt table columns and CSV parsers. Markdown blocks are immune to delimiters.
4. **Machine and Human Ergonomics:**
   Both LLMs and human engineers can read, generate, and edit Structured Markdown Blocks without horizontal scrolling or character-count constraints.

---

## 2. Phase 1: Checklist Standards

### 2.1 File Location & Naming Conventions

All Phase 1 Checklists must be placed in:
`docs/<module>/<feature>/checklist.md`

Examples:
- `docs/auth/login/checklist.md`
- `docs/chat/attachment-upload/checklist.md`
- `docs/billing/checkout/checklist.md`

### 2.2 Exact Template Syntax

Every `checklist.md` must adhere to this exact structural format:

```markdown
F: <Feature Name>

# <Category 1>
## <Subcategory 1.1>
- [P1][@e2e] CHK-<FEATURE>-001: <Observable scenario check description>
- [P2][@api] CHK-<FEATURE>-002: <Observable scenario check description>

## <Subcategory 1.2>
- [P2][@e2e] CHK-<FEATURE>-003: <Observable scenario check description>
- [P3][@ui] CHK-<FEATURE>-004: <Observable scenario check description>

# <Category 2>
## <Subcategory 2.1>
- [P1][@api] CHK-<FEATURE>-005: <Observable scenario check description>
```

### 2.3 The 4 Core Quality Criteria

#### Criteria 1: MECE (Mutually Exclusive, Collectively Exhaustive)
- **Mutually Exclusive:** No two check items verify the same risk or functional path. Overlapping checks waste execution time and complicate root-cause analysis.
- **Collectively Exhaustive:** All relevant quality dimensions must be accounted for:
  - Happy Path / Core Flows
  - Input Validation & Boundary Limits
  - Security & Permission Boundaries (Role-Based Access Control)
  - Network & Service Failure Modes
  - Data Persistence & Real-time State Synchronization

#### Criteria 2: Atomic Scenario
- **1 Check Item = 1 Scenario Verifying 1 Observable Business Outcome.**
- **Anti-Pattern (Forbidden):** Micro-element inspection checks:
  - `[FAIL]` `- [P3][@ui] CHK-ATTACH-001: Verify file upload button has blue border and 12px padding`
  - `[FAIL]` `- [P3][@ui] CHK-ATTACH-002: Verify file picker opens on click`
- **Best Practice (Allowed):** Complete action-feedback loops:
  - `[PASS]` `- [P1][@e2e] CHK-CHAT-ATTACH-001: Send image attachment with valid PNG and verify message renders in chat with thumbnail preview`

#### Criteria 3: Explicit Metadata
Every line item must define two mandatory bracketed tags:
1. **Priority Tag (`[P1..P4]`):**
   - `[P1]` **Critical:** Primary revenue flows, authentication, checkout, smoke tests. System is unusable if this fails.
   - `[P2]` **High:** Core feature capabilities, standard business rules, critical error feedback.
   - `[P3]` **Medium:** Secondary validations, edge cases, edge data formats, non-blocking UI states.
   - `[P4]` **Low:** Minor polish, cosmetic edge cases, low-probability scenarios.
2. **Layer Tag (`[@layer]`):**
   - `[@api]`: Headless HTTP / WebSocket endpoint testing via API client.
   - `[@e2e]`: Full end-to-end browser journey using Playwright.
   - `[@ui]`: Browser-based client-side component/layout validation.
   - `[@unit]`: Pure function helpers in `src/utils/` (executed via Vitest).

#### Criteria 4: Immutable ID
- Format: `CHK-<FEATURE>-<NNN>` (e.g., `CHK-CHAT-ATTACH-001`, `CHK-AUTH-005`).
- Generated upfront in Phase 1.
- **Rules of Immutability:**
  - An ID assigned to a check item remains permanently bound to that scenario.
  - Never renumber IDs when items are inserted, deleted, or reordered.
  - If an item is deprecated, mark it deprecated rather than reassigning its number.
  - Ensures seamless traceability across LLM prompts, testcases, git commit history, and test logs.

### 2.4 Structural Hierarchy & Ordering Rules

1. **Ordering by User Flow:** Order categories and subcategories chronologically following the end-to-end user journey (e.g., Initiation -> Validation -> Processing -> Real-time Propagation -> Cleanup).
2. **Internal Item Ordering:** Within each subcategory, order items systematically:
   - Happy paths first (`[P1]`, `[P2]`)
   - Negative validation and boundary cases next (`[P2]`, `[P3]`)
   - Edge cases, security, and concurrency last (`[P3]`, `[P4]`)
3. **Group by Region / Workflow:** Avoid element-by-element checklists. Group by functional region or interaction pattern.
4. **Exclude Generic Chrome:** Do not write checklist items for standard application navigation bars, headers, or footers unless the feature explicitly alters them.

### 2.5 Realistic Production Example: `checklist.md`

Below is an authentic specification for the **Chat Message Attachment & Upload** feature:

```markdown
F: Chat Message Attachment & Upload

# Attachment Upload
## File Type & Extension Validation
- [P1][@e2e] CHK-CHAT-ATTACH-001: Upload valid PNG image within size limit and verify message renders with preview thumbnail
- [P2][@e2e] CHK-CHAT-ATTACH-002: Upload valid PDF document and verify message renders with file icon, filename, and download button
- [P2][@e2e] CHK-CHAT-ATTACH-003: Upload unsupported file executable (.exe) and verify rejection with unsupported format error toast
- [P3][@api] CHK-CHAT-ATTACH-004: Direct API upload with spoofed MIME type and verify HTTP 415 Unsupported Media Type rejection

## File Size Boundaries & Quota
- [P1][@e2e] CHK-CHAT-ATTACH-005: Upload file exceeding 25MB ceiling and verify client-side validation toast without initiating upload request
- [P2][@api] CHK-CHAT-ATTACH-006: Direct API multipart upload exceeding 25MB payload and verify HTTP 413 Payload Too Large response
- [P3][@e2e] CHK-CHAT-ATTACH-007: Upload zero-byte empty file and verify validation rejection error

# Multi-Actor Realtime Synchronization
## Concurrency & Delivery
- [P1][@e2e] CHK-CHAT-ATTACH-008: Sender uploads image in active channel; verify Receiver receives real-time WebSocket event and renders image thumbnail within 2s
- [P2][@e2e] CHK-CHAT-ATTACH-009: Sender aborts in-progress large file upload; verify no message appears in Receiver channel

# Permissions & Cleanup
## Role-Based Deletion
- [P2][@e2e] CHK-CHAT-ATTACH-010: Standard user deletes their own uploaded attachment; verify attachment is removed for all participants
- [P3][@api] CHK-CHAT-ATTACH-011: Standard user attempts to delete attachment uploaded by another user; verify HTTP 403 Forbidden
```

### 2.6 Checklist Quality Gate

Before declaring Phase 1 complete, verify:
- [ ] Every item has an immutable ID in format `CHK-<FEATURE>-<NNN>`.
- [ ] Every item has a priority `[P1..P4]` and layer `[@api|@e2e|@ui|@unit]`.
- [ ] Checks focus on business outcome (WHAT), not selector details (HOW).
- [ ] Scenarios are atomic and MECE.

---

## 3. Phase 2: Test Case Standards

### 3.1 File Location & Naming Conventions

All Phase 2 Test Case specifications must be placed in:
`docs/<module>/<feature>/testcases.md`

Examples:
- `docs/auth/login/testcases.md`
- `docs/chat/attachment-upload/testcases.md`
- `docs/billing/checkout/testcases.md`

### 3.2 Exact Template Syntax (Structured Markdown Blocks)

Every test case in `testcases.md` must be written in the following Structured Markdown Block format:

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

### 3.3 The 5 Core Quality Criteria

#### Criteria 1: Determinism & Unambiguity
- **Eliminate subjective adjectives:** Banish words like "quickly", "properly", "correctly", "smoothly", "instantly".
- **Quantitative thresholds:** Use concrete numbers: `< 2s response`, `HTTP 200 OK`, `HTTP 413 Payload Too Large`.
- **Standardized DOM states:** Express expectations in verifiable assertions: `Visible`, `Hidden`, `Disabled`, `Enabled`, `Count = 1`.

#### Criteria 2: Actors & Roles
- Explicitly declare all user personas participating in the test.
- Every role must map to a pre-authenticated session state managed in `src/fixtures/auth.fixture.ts` (e.g., `authFile('user')`, `authFile('admin')`).
- Multi-Actor support: Real-time chat, approval workflows, and collaborative editing require multiple distinct browser contexts running simultaneously (Actor 1 vs Actor 2).

#### Criteria 3: API Seeding & Teardown
- **API-First Setup:** Never use UI clicks to log in or create prerequisite test data. Always seed prerequisite entities (channels, users, items) via `src/api/` clients using `APIRequestContext`.
- **Mandatory API Teardown:** Every test case must declare explicit teardown logic in its Post-conditions block (or rely on self-cleaning janitor fixtures). Tests must never leave dirty state in shared databases or workspaces.
- **Parallel Worker Safety:** With 10 workers executing concurrently (`fullyParallel: true`), tests must not mutate shared records.

#### Criteria 4: Verifiable Assertions & UI Copy Constants
- **Verbatim Error Constants:** UI validation messages, system toasts, and modal descriptions must reference constants from `src/constants/error-messages.ts` (`UI_MESSAGES.*`).
- Never paraphrase UI copy in test cases. If the UI says *"File size exceeds 25MB limit"*, the testcase must not state *"Verify error tells user file is too big"*.

#### Criteria 5: Data Isolation
- Follow the mandatory naming convention from `CLAUDE.md:69`:
  `qa_auto_<workerIndex>_<uuid>`
- **Never use `Date.now()`:** Millisecond timestamps collide across parallel worker processes and lack identifiable ownership.
- The `qa_auto_` prefix ensures orphan data can be identified and purged safely by background sweeps.

### 3.4 Multi-Actor Concurrency in Real-Time Systems

In messaging and collaborative applications, real-time message delivery is a primary failure vector. Specifications must define multi-actor interaction patterns cleanly:

- **Actor 1 (Primary / Sender):** Interacts with the active UI (types message, uploads attachment).
- **Actor 2 (Secondary / Receiver):** Observes real-time WebSocket state in a separate isolated browser context.
- **Synchronization Point:** Explicitly specify what Actor 2 awaits (e.g., incoming message bubble, download link, preview thumbnail) with a quantitative timeout limit (`< 2000ms`).

---

### 3.5 Realistic Production Examples: `testcases.md`

#### Example 1: Multi-Actor Chat File Upload (`TC-CHAT-ATTACH-001`)

```markdown
### TC-CHAT-ATTACH-001: Upload valid PNG image and verify real-time thumbnail delivery
- **Trace:** CHK-CHAT-ATTACH-001 | **Priority:** [P1] | **Layer:** @e2e | **Suite:** smoke | **Role:** user
- **Actors:**
  - Actor 1 (Primary - Sender): Role `user` (storageState: `authFile('user')`)
  - Actor 2 (Secondary - Receiver): Role `leader` (storageState: `authFile('leader')`)
- **Pre-conditions (API Seeding):**
  - Chat channel created via `chatApi.createChannel({ name: 'qa_auto_${workerIndex}_${uuid}', members: ['user', 'leader'] })`.
  - Both actors navigate to `/channels/${channelId}`.
- **Test Data:**
  - File: `test-data/sample-files/sample-image.png` (PNG, 512 KB, 800x600 px).
- **Steps:**
  1. **Action:** Actor 1 clicks the attachment icon in the message compose box and selects `test-data/sample-files/sample-image.png`.
     **Expected:** File upload staging container displays the image name `sample-image.png` and progress bar reaches `100%`. The "Send" button becomes `Enabled`.
  2. **Action:** Actor 1 clicks the "Send" button.
     **Expected:** Compose input is cleared. An outbound message item renders in Actor 1's timeline with state `Sent` and image thumbnail is `Visible`.
  3. **Action:** Actor 2 observes the active chat timeline.
     **Expected:** Inbound message appears in Actor 2's timeline within 2000ms via WebSocket. Message contains sender label `user`, thumbnail image is `Visible`, and clicking the thumbnail opens the lightbox viewer.
- **Post-conditions (API Teardown):**
  - Delete created channel via `chatApi.deleteChannel(channelId)`.
  - Delete uploaded file object via `storageApi.deleteFile(attachmentId)`.
```

#### Example 2: Negative Validation & Error Contract (`TC-CHAT-ATTACH-005`)

```markdown
### TC-CHAT-ATTACH-005: Upload file exceeding 25MB ceiling displays client-side error toast
- **Trace:** CHK-CHAT-ATTACH-005 | **Priority:** [P1] | **Layer:** @e2e | **Suite:** regression | **Role:** user
- **Actors:**
  - Actor 1 (Primary): Role `user` (storageState: `authFile('user')`)
- **Pre-conditions (API Seeding):**
  - Direct channel initialized via `chatApi.createChannel({ name: 'qa_auto_${workerIndex}_${uuid}' })`.
  - Actor 1 navigates to `/channels/${channelId}`.
- **Test Data:**
  - File: `test-data/sample-files/oversized-archive.zip` (26.2 MB binary).
  - Error copy constant: `UI_MESSAGES.CHAT.FILE_SIZE_EXCEEDED` ("Maximum file size allowed is 25MB.").
- **Steps:**
  1. **Action:** Actor 1 drops `test-data/sample-files/oversized-archive.zip` onto the chat drop-zone area.
     **Expected:** Drop-zone overlay disappears. No network POST request is dispatched to `/api/v1/storage/upload`.
  2. **Action:** Observe notification toast container.
     **Expected:** Toast notification appears within 500ms with severity `Error`, displaying exact text matching `UI_MESSAGES.CHAT.FILE_SIZE_EXCEEDED`. The file is not added to the compose staging area.
- **Post-conditions (API Teardown):**
  - Delete created channel via `chatApi.deleteChannel(channelId)`.
```

---

## 4. Phase 3: Playwright Automation Mapping Rules

### 4.1 Specification-to-Code Mapping Reference

Every structural element in `testcases.md` maps directly to standard `@playwright/test` constructs:

| Test Case Spec Element | Playwright Test Construct | Implementation Rule |
| :--- | :--- | :--- |
| `# Category` | `test.describe('Category', () => { ... })` | Outer test suite grouping |
| `## Subcategory` | `test.describe('Subcategory', () => { ... })` | Inner test suite grouping |
| `TC-<FEATURE>-<NNN>: <Title>` | `test('TC-<FEATURE>-<NNN>: <Title>', ...)` | Unique test name preserving traceability |
| Priority `[P1]` & Layer `@e2e` | `{ tag: ['@p1', '@e2e', '@smoke'] }` | Playwright test tags for CLI filtering (`--grep`) |
| **Actors** (Multi-Actor) | Custom fixtures or `browser.newContext()` | Authenticated contexts via `authFile('<role>')` |
| **Pre-conditions (API Seeding)** | Test setup via `src/api/` clients | `test.beforeEach` or dedicated seeding fixture |
| **Steps** (`1. Action ... Expected`) | `await test.step('1. Action ...', async () => {})` | Each step wrapped in Playwright `test.step` |
| **Verifiable Assertions** | Web-first `expect(locator)` | Auto-retrying assertions (`toBeVisible`, `toHaveText`) |
| **Post-conditions (API Teardown)** | Self-cleaning fixtures or `test.afterEach` | API cleanup executing in `finally` blocks |

### 4.2 Complete TypeScript Automation Spec

The following production code demonstrates the exact implementation of `TC-CHAT-ATTACH-001` in Playwright:

```typescript
// tests/regression/chat/chat-attachment.spec.ts
import { test, expect } from '@playwright/test';
import { randomUUID } from 'node:crypto';
import { authFile } from '../../../src/fixtures/auth.fixture';
import { ChatPage } from '../../../src/pages/chat/chat.page';
import { ChatApiClient } from '../../../src/api/chat.api';

test.describe('Attachment Upload', () => {
  test.describe('File Type & Extension Validation', () => {
    let chatApi: ChatApiClient;
    let channelId: string;

    test.beforeEach(async ({ request }, testInfo) => {
      chatApi = new ChatApiClient(request);
      const uniqueChannelName = `qa_auto_${testInfo.workerIndex}_${randomUUID()}`;
      const channel = await chatApi.createChannel({
        name: uniqueChannelName,
        members: ['user', 'leader'],
      });
      channelId = channel.id;
    });

    test.afterEach(async () => {
      if (channelId) {
        await chatApi.deleteChannel(channelId);
        // Note: storageApi.deleteFile(attachmentId) cleanup handled here if attachment is not cascaded
      }
    });

    test(
      'TC-CHAT-ATTACH-001: Upload valid PNG image and verify real-time thumbnail delivery',
      { tag: ['@p1', '@e2e', '@smoke'] },
      async ({ browser }) => {
        // Multi-Actor Context Initialization
        const senderContext = await browser.newContext({
          storageState: authFile('user'),
        });
        const receiverContext = await browser.newContext({
          storageState: authFile('leader'),
        });

        const senderPage = new ChatPage(await senderContext.newPage());
        const receiverPage = new ChatPage(await receiverContext.newPage());

        try {
          await test.step('Pre-condition: Navigate actors to target channel', async () => {
            await Promise.all([
              senderPage.goto(channelId),
              receiverPage.goto(channelId),
            ]);
          });

          await test.step('1. Sender selects and uploads valid PNG attachment', async () => {
            const filePath = 'test-data/sample-files/sample-image.png';
            await senderPage.attachFile(filePath);

            await expect(senderPage.uploadProgressBar).toHaveAttribute('data-progress', '100');
            await expect(senderPage.sendButton).toBeEnabled();
          });

          await test.step('2. Sender dispatches message and verifies local thumbnail', async () => {
            await senderPage.sendButton.click();

            await expect(senderPage.messageInput).toBeEmpty();
            const sentMessage = senderPage.getLatestMessage();
            await expect(sentMessage.thumbnail).toBeVisible();
          });

          await test.step('3. Receiver verifies real-time thumbnail delivery within 2s', async () => {
            const receivedMessage = receiverPage.getLatestMessage();
            await expect(receivedMessage.root).toBeVisible({ timeout: 2000 });
            await expect(receivedMessage.senderName).toHaveText('user');
            await expect(receivedMessage.thumbnail).toBeVisible();
          });
        } finally {
          await senderContext.close();
          await receiverContext.close();
        }
      }
    );
  });
});
```

### 4.3 Clean Code & Anti-Flakiness Rules in Generated Tests

When translating test cases into automation code, apply the repository invariants:
1. **Never use `page.waitForTimeout()` or artificial sleeps:** Always rely on web-first assertions (`expect(locator).toBeVisible()`) which automatically retry until the timeout.
2. **Setup request promises before triggering actions:** When verifying network responses, create `page.waitForResponse(...)` *before* clicking the button that triggers the request.
3. **Keep locators in Page Objects (`src/pages/`):** Specs must never contain raw CSS or XPath selectors. Locators and user actions belong inside POM classes. Specs only contain flow logic and assertions.
4. **Always clean up browser contexts and test records:** Wrap multi-actor contexts in `try ... finally` blocks to prevent resource leaks during CI test retries.

---

## 5. Anti-Patterns & Common Traps

### 5.1 Checklist Anti-Patterns

| Anti-Pattern | Bad Example | Corrected Best Practice | Rationale |
| :--- | :--- | :--- | :--- |
| **Micro-Element Inspection** | `- [P3][@ui] CHK-001: Verify avatar icon is circular with 32px height` | `- [P2][@ui] CHK-001: Verify user profile header displays avatar and user handle` | Checklists map business risk, not CSS style sheet declarations. |
| **Missing Metadata Tags** | `- Check user can login with valid email` | `- [P1][@e2e] CHK-AUTH-001: User logs in with valid credentials and reaches dashboard` | Without priority and layer tags, automated test generation and test sharding cannot function. |
| **Mutable / Floating IDs** | `- CHK-1: ...`<br>`- CHK-2: ...` (renumbered on edit) | `- CHK-AUTH-001: ...`<br>`- CHK-AUTH-002: ...` (immutable) | Renumbering IDs breaks traceability with `testcases.md` and git history. |
| **Vague Business Verifications** | `- [P1][@e2e] CHK-005: Check file upload works` | `- [P1][@e2e] CHK-CHAT-ATTACH-001: Upload valid PNG image within size limit and verify thumbnail in timeline` | Specific scenario boundaries are mandatory to generate deterministic test steps. |

### 5.2 Test Case Anti-Patterns

| Anti-Pattern | Bad Example | Corrected Best Practice | Rationale |
| :--- | :--- | :--- | :--- |
| **Subjective Assertions** | `**Expected:** System responds quickly and file appears properly.` | `**Expected:** Message appears in timeline within 2000ms with state 'Delivered'.` | Automated test engines cannot assert on "quickly" or "properly". |
| **UI-Driven Data Seeding** | `**Pre-conditions:** Log into app via UI, click Settings, click Create Channel, fill form, submit.` | `**Pre-conditions (API Seeding):** Channel created via chatApi.createChannel({ name: 'qa_auto_${workerIndex}_${uuid}' })` | UI-based seeding is slow, flaky, and creates cascade failures if login UI breaks. |
| **Missing API Teardown** | `**Post-conditions:** None` | `**Post-conditions (API Teardown):** Delete channel via chatApi.deleteChannel(channelId)` | Leftover test data causes data pollution and quota errors across 10 parallel CI workers. |
| **Inlined String Paraphrasing** | `**Expected:** Toast says file is too big to upload.` | `**Expected:** Toast renders UI_MESSAGES.CHAT.FILE_SIZE_EXCEEDED ("Maximum file size allowed is 25MB.").` | Inlined paraphrases break when UI copy changes; exact constants guarantee maintainability. |
| **Timestamp Collisions** | `const name = 'test_' + Date.now();` | `const name = 'qa_auto_' + workerIndex + '_' + randomUUID();` | Parallel workers running simultaneously generate identical millisecond timestamps. |

---

## 6. Summary Authoring Checklist

### Checklist Authoring Gate (Phase 1)
- [ ] Requirements fully analyzed from PRD, API docs, or screen captures.
- [ ] Hierarchy structured using `# Category` and `## Subcategory`.
- [ ] Every item has an immutable ID: `CHK-<FEATURE>-<NNN>`.
- [ ] Every item specifies priority tag `[P1]`, `[P2]`, `[P3]`, or `[P4]`.
- [ ] Every item specifies layer tag `[@api]`, `[@e2e]`, `[@ui]`, or `[@unit]`.
- [ ] Scenarios are atomic, MECE, and describe observable business outcomes.
- [ ] Happy path listed first, followed by validation and boundary errors.

### Test Case Authoring Gate (Phase 2)
- [ ] Every test case maps 1-to-1 to a checklist item: `TC-<FEATURE>-<NNN>` traces `CHK-<FEATURE>-<NNN>`.
- [ ] Formatted as Structured Markdown Blocks (no Markdown tables or CSV).
- [ ] Explicit **Actors** defined with `storageState` roles (`authFile('<role>')`).
- [ ] **Pre-conditions** use API seeding with `qa_auto_<workerIndex>_<uuid>` data isolation.
- [ ] **Test Data** references paths in `test-data/` or constants in `src/constants/`.
- [ ] **Steps** define concrete **Action** and verifiable **Expected** DOM states or HTTP codes.
- [ ] Assertions use verbatim constants from `src/constants/error-messages.ts`.
- [ ] **Post-conditions** declare mandatory API cleanup.

### Automation Code Gen Gate (Phase 3)
- [ ] Category and subcategory map to `test.describe()`.
- [ ] Test case title, ID, and tags (`@p1`, `@e2e`) map to `test()`.
- [ ] Steps wrapped in `await test.step()`.
- [ ] Pre-conditions and teardown implemented via `src/api/` clients.
- [ ] Locators strictly encapsulated inside `src/pages/` Page Object Models.
- [ ] Zero arbitrary timeouts (`page.waitForTimeout`); web-first assertions only.
- [ ] Test executes and passes deterministically under 10 parallel workers.
