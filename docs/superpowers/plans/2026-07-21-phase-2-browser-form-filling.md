# Phase 2 Browser Form Filling Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Chrome extension that safely inspects, plans, fills, verifies, and reviews application forms without ever submitting them.

**Architecture:** Keep page inspection and DOM actions inside a Manifest V3 content script. The extension sends normalized field descriptions to the loopback Core, receives a typed fill plan, executes one field at a time, and persists checkpoints after every page.

**Tech Stack:** TypeScript, Chrome Manifest V3, Vite, React, Zod, Vitest, Playwright, shared workspace contracts.

---

## File map

- `apps/extension/src/content/*`: DOM inspection, safe field actions, verification.
- `apps/extension/src/background/*`: loopback communication and tab lifecycle.
- `apps/extension/src/sidepanel/*`: review UI.
- `apps/core/src/form-plans/*`: semantic mapping, risk policy, checkpoints.
- `packages/contracts/src/forms.ts`: cross-boundary messages.

### Task 1: Define form contracts and risk policy

**Files:** Create `packages/contracts/src/forms.ts`; modify `packages/contracts/src/index.ts`; create `apps/core/src/form-plans/risk-policy.ts`; test `apps/core/src/form-plans/risk-policy.test.ts`.

- [ ] Write a failing table test proving `family_relation`, `compliance_attestation`, `id_number`, and `voluntary_adjustment` always return `leave_blank`, regardless of confidence.
- [ ] Run `pnpm --filter @auto-find-job/core test -- risk-policy.test.ts`; expect FAIL.
- [ ] Implement `PageField`, `FillInstruction`, `FillPlan`, `FillResult`, and `Checkpoint` Zod schemas plus:

```ts
export function decideAction(risk: Risk, confidence: number) {
  if (risk === "high") return "leave_blank" as const;
  if (confidence >= 0.9) return "fill" as const;
  if (confidence >= 0.65) return "fill_and_review" as const;
  return "leave_blank" as const;
}
```

- [ ] Run the focused test and `pnpm typecheck`; expect PASS.
- [ ] Commit with `git commit -am "feat: define safe form fill contracts"` after staging new files.

### Task 2: Inspect standard form controls

**Files:** Create `apps/extension/manifest.json`, `apps/extension/src/content/inspect-page.ts`, `apps/extension/src/content/labels.ts`; test `apps/extension/src/content/inspect-page.test.ts`.

- [ ] Write jsdom tests for text inputs, textareas, native select, radio groups, checkboxes, file inputs, required flags, `aria-label`, associated labels, and nearby text.
- [ ] Run `pnpm --filter @auto-find-job/extension test -- inspect-page.test.ts`; expect FAIL.
- [ ] Implement stable `fieldId` from DOM path plus semantic attributes. Ignore hidden, disabled, password, CSRF, submit, and reset controls. Never serialize current password or cookie values.
- [ ] Run tests; expect all fixtures normalized into `PageField[]`.
- [ ] Commit `feat: inspect application form controls`.

### Task 3: Build deterministic field executors

**Files:** Create `apps/extension/src/content/actions.ts`, `apps/extension/src/content/verify.ts`; test `apps/extension/src/content/actions.test.ts`.

- [ ] Write failing tests that exercise input, textarea, select, radio, checkbox, date, and file instructions and prove `button[type=submit]` is rejected.
- [ ] Run the focused test; expect FAIL.
- [ ] Implement one action per control type using native setters and `input`/`change` events. Restrict clicks to an allowlist of non-submit targets. After each action, read the effective value and return `verified`, `mismatch`, or `page_error`.
- [ ] Add an AST/static test that scans extension sources and fails on `.submit(`, `requestSubmit(`, or clicks targeting submit controls.
- [ ] Run tests; expect PASS.
- [ ] Commit `feat: execute and verify safe form actions`.

### Task 4: Map page fields to facts in Core

**Files:** Create `apps/core/src/form-plans/dictionary.ts`, `mapper.ts`, `plan-service.ts`; test `apps/core/src/form-plans/plan-service.test.ts`.

- [ ] Write tests for Chinese synonyms such as `院校/学校名称`, `专业/所学专业`, `最高学历/当前学历`, including ambiguous and high-risk cases.
- [ ] Run focused tests; expect FAIL.
- [ ] Implement deterministic dictionary matching first. Only unmatched low-risk labels may use the AI provider; AI output still passes risk policy and must cite a confirmed fact ID. Never send current page values classified personal/restricted.
- [ ] Run tests; expect ambiguous mappings marked for review and high-risk mappings blank.
- [ ] Commit `feat: map application fields to verified facts`.

### Task 5: Add authenticated Core-extension messaging and checkpoints

**Files:** Create `apps/extension/src/background/core-client.ts`, `apps/core/src/routes/form-plans.ts`, `apps/core/src/form-plans/checkpoint-repository.ts`; tests beside each file.

- [ ] Write tests for missing token, disallowed origin, stale checkpoint version, and successful plan/result exchange.
- [ ] Run relevant tests; expect FAIL.
- [ ] Implement loopback-only requests with install token, explicit CORS allowlist for the extension ID, application ID, page URL, and monotonically increasing checkpoint version. Save results after each page.
- [ ] Run tests; expect PASS and stale writes return HTTP 409.
- [ ] Commit `feat: connect extension with resumable form plans`.

### Task 6: Build side-panel review and end-to-end fixture tests

**Files:** Create `apps/extension/src/sidepanel/App.tsx`, `FieldReviewList.tsx`, `apps/extension/e2e/fixtures/application.html`, `apps/extension/e2e/fill.spec.ts`.

- [ ] Write a Playwright test that loads the extension and fixture, fills green/yellow fields, leaves red fields blank, uploads a fixture PDF, reloads, and resumes from checkpoint.
- [ ] Run `pnpm --filter @auto-find-job/extension test:e2e`; expect FAIL.
- [ ] Implement the side panel with green/yellow/red groups, source fact links, before/after values, retry-current-page, and “mark submitted” button. Do not render a submit action.
- [ ] Run `pnpm test && pnpm typecheck && pnpm --filter @auto-find-job/extension build && pnpm --filter @auto-find-job/extension test:e2e`; expect PASS.
- [ ] Commit `feat: deliver reviewed browser form filling`.

## Phase 2 acceptance

Use the fixture form to verify at least 90% of standard fields are filled and read back correctly, all high-risk fields remain blank, failed uploads block the ready state, reload resumes from the last checkpoint, and the built extension contains no final-submit code path.

