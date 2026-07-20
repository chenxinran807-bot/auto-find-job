# Phase 4 Tracking and Reliability Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver the application dashboard, resumable orchestration, privacy-safe diagnostics, backup, deletion, and full acceptance gates.

**Architecture:** Model applications as an explicit transition-checked state machine. Store task attempts and field results append-only, expose operational health in the console, and use redacted exports for diagnostics.

**Tech Stack:** TypeScript, Fastify, SQLite/Drizzle, React, TanStack Query, Vitest, Playwright.

---

## File map

- `apps/core/src/workflow/*`: state machine, queue, retries, recovery.
- `apps/core/src/audit/*`: redaction and diagnostic bundles.
- `apps/core/src/backup/*`: encrypted export/import and deletion.
- `apps/console/src/features/dashboard/*`: pipeline and attention inboxes.
- `apps/console/e2e/*`: end-to-end acceptance.

### Task 1: Enforce application transitions and idempotent tasks

**Files:** Create `apps/core/src/workflow/state-machine.ts`, `task-repository.ts`, `runner.ts`; tests beside files.

- [ ] Write a transition table test covering `discovered → scored → shortlisted → package_ready → form_in_progress → review_required → submitted` and rejecting skipped/backward transitions.
- [ ] Write an idempotency test proving the same task key cannot create a second package or duplicate form run.
- [ ] Run workflow tests; expect FAIL.
- [ ] Implement explicit transition map, `blockedReason` orthogonal to status, lease-based task claims, bounded retries, and exponential backoff with jitter.
- [ ] Run tests; expect PASS, including simulated process restart.
- [ ] Commit `feat: add resumable application workflow`.

### Task 2: Build candidate and review inboxes

**Files:** Create `apps/console/src/features/dashboard/Pipeline.tsx`, `CandidateInbox.tsx`, `ReviewInbox.tsx`, `ApplicationDetail.tsx`; tests beside components.

- [ ] Write UI tests for sorting by match/deadline, bulk shortlist,志愿顺序 conflict warning, red/yellow/green field groups, and package version display.
- [ ] Run console tests; expect FAIL.
- [ ] Implement accessible tables/cards with URL-addressable filters. Bulk actions may generate packages but never submit applications. Application detail must show JD snapshot, claims with sources, model/prompt versions, attempts, and blockers.
- [ ] Run console tests and build; expect PASS.
- [ ] Commit `feat: add application workflow dashboard`.

### Task 3: Add privacy-safe audit and diagnostics

**Files:** Create `apps/core/src/audit/redactor.ts`, `event-writer.ts`, `diagnostic-bundle.ts`; test `redactor.test.ts` and `diagnostic-bundle.test.ts`.

- [ ] Write adversarial tests containing phone, email, ID number, token, cookie, Authorization header, resume text, and file paths.
- [ ] Run audit tests; expect FAIL.
- [ ] Implement structured allowlist logging. Replace restricted values with typed markers, hash local identifiers with an install salt, and reject arbitrary object logging. Diagnostic bundles include versions, redacted field labels, adapter/source health, and errors—never page values or documents.
- [ ] Run audit tests; expect PASS with no secret fixture substrings in output.
- [ ] Commit `feat: add redacted workflow diagnostics`.

### Task 4: Implement backup, restore, and complete local deletion

**Files:** Create `apps/core/src/backup/export.ts`, `import.ts`, `delete-profile.ts`; create `apps/console/src/features/settings/DataControls.tsx`; tests beside files.

- [ ] Write tests for round-trip export/import, wrong passphrase, schema version mismatch, and deletion of facts, applications, generated packages, indexes, API keys, and audit references.
- [ ] Run backup tests; expect FAIL.
- [ ] Implement password-encrypted archive export with manifest and hashes. Restore into a temporary database, validate fully, then atomically replace. Deletion requires exact profile-name confirmation and produces a non-sensitive completion receipt.
- [ ] Run tests; expect PASS and no deleted identifiers remain in database or package directories.
- [ ] Commit `feat: add secure data portability controls`.

### Task 5: Add operational health and recovery UI

**Files:** Create `apps/core/src/routes/health.ts`, `apps/console/src/features/settings/SystemHealth.tsx`; tests beside files.

- [ ] Write tests for model unavailable, source degraded, extension disconnected, migration required, disk unwritable, and healthy state.
- [ ] Run health tests; expect FAIL.
- [ ] Implement actionable health cards with last success, safe retry, and documentation link. Never auto-retry high-risk form actions; retry only idempotent discovery/generation tasks.
- [ ] Run tests; expect PASS.
- [ ] Commit `feat: surface workflow health and recovery`.

### Task 6: Run full acceptance and release checks

**Files:** Create `apps/console/e2e/full-workflow.spec.ts`, `scripts/check-no-submit.mjs`, `docs/qa/release-checklist.md`.

- [ ] Write an e2e test using a fake AI provider and fixture source: import → confirm → discover → score → shortlist → package → fill → review → user-mark-submitted → dashboard.
- [ ] Implement `check-no-submit.mjs` to scan built extension JavaScript for prohibited submit APIs and submit-target clicks; make it part of `pnpm verify`.
- [ ] Run `pnpm verify`; expected: unit, integration, e2e, typecheck, lint, builds, migration check, secret scan, and no-submit scan all PASS.
- [ ] Execute the seven-site rehearsal checklist from Phase 3 against the release build; expected: ≥90% verified standard-field accuracy, zero high-risk guesses, zero automatic submissions.
- [ ] Commit `test: add end-to-end release gates`.

## Phase 4 acceptance

The full local workflow survives a process restart, shows actionable blockers, produces redacted diagnostics, supports verified backup/restore and complete deletion, and passes `pnpm verify`. A release is blocked if any high-risk field was guessed, any unsupported claim was generated, any required attachment failed, or any final-submit code path exists.

