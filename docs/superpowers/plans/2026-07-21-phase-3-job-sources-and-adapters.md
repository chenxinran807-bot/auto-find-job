# Phase 3 Job Sources and Site Adapters Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Discover and normalize campus jobs from seven target companies and add narrowly scoped site adapters based on observed failures.

**Architecture:** Each job source and site adapter implements a versioned contract. Public endpoints are used only when permitted; otherwise the extension captures a job the user has opened. Saved, redacted fixtures drive contract tests and detect page changes.

**Tech Stack:** TypeScript, Fastify, Playwright, Cheerio, Vitest, Chrome extension contracts, SQLite.

---

## File map

- `packages/contracts/src/jobs.ts`, `adapters.ts`: plugin contracts.
- `apps/core/src/job-sources/*`: scheduler, normalization, dedupe, source plugins.
- `apps/extension/src/adapters/*`: special controls only.
- `fixtures/sites/{bytedance,kimi,meituan,pdd,baidu,kuaishou,xiaohongshu}/*`: redacted approved fixtures and metadata.

### Task 1: Define plugin contracts and canonical job identity

**Files:** Create `packages/contracts/src/jobs.ts`, `packages/contracts/src/adapters.ts`, `apps/core/src/job-sources/fingerprint.ts`; test `fingerprint.test.ts`.

- [ ] Write failing tests proving the same company/role/location from two URLs shares a fingerprint while materially different roles do not.
- [ ] Run focused test; expect FAIL.
- [ ] Implement normalized Unicode, whitespace, company alias, role, city, and stable upstream ID hashing. Define `discover`, `fetchDetails`, `normalize`, and `fingerprint` source methods plus `detect`, `inspect`, `plan`, `execute`, and `verify` adapter methods.
- [ ] Run tests and typecheck; expect PASS.
- [ ] Commit `feat: define job source and site adapter contracts`.

### Task 2: Build respectful source runner and browser-capture fallback

**Files:** Create `apps/core/src/job-sources/runner.ts`, `rate-limit.ts`, `repository.ts`; create `apps/extension/src/content/capture-job.ts`; tests beside files.

- [ ] Write tests for per-host concurrency 1, configurable delay, retry-after handling, duplicate merge, removed-job retention, and user-opened page capture.
- [ ] Run focused tests; expect FAIL.
- [ ] Implement source policy states `automated`, `manual_capture`, and `disabled`. Runner must never bypass authentication, CAPTCHA, robots/policy rejection, or access controls; it switches to manual capture and records the reason.
- [ ] Run tests; expect PASS.
- [ ] Commit `feat: add policy-aware job discovery runner`.

### Task 3: Implement seven source plugins from approved fixtures

**Files:** Create one folder each under `apps/core/src/job-sources/providers/`: `bytedance`, `kimi`, `meituan`, `pdd`, `baidu`, `kuaishou`, `xiaohongshu`; create matching redacted fixtures under `fixtures/sites`.

- [ ] For each company, save only a permitted public response or user-opened redacted page fixture plus `fixture-meta.json` containing source URL, capture date, and redaction note.
- [ ] Add a failing contract test per provider asserting title, company, locations, category, JD, URL, upstream ID, and update timestamp.
- [ ] Implement the smallest parser that passes its fixture. If automated retrieval is disallowed or unstable, implement `manual_capture` rather than stealth logic.
- [ ] Run `pnpm --filter @auto-find-job/core test -- job-sources`; expect seven provider suites PASS.
- [ ] Commit providers separately with these messages: `feat: add bytedance campus job source`, `feat: add kimi campus job source`, `feat: add meituan campus job source`, `feat: add pdd campus job source`, `feat: add baidu campus job source`, `feat: add kuaishou campus job source`, and `feat: add xiaohongshu campus job source`.

### Task 4: Add source health and change detection

**Files:** Create `apps/core/src/job-sources/health.ts`, `apps/core/src/routes/source-health.ts`; test `health.test.ts`.

- [ ] Write tests for empty discovery, parse-error spike, required-field loss, and healthy runs.
- [ ] Run focused tests; expect FAIL.
- [ ] Implement run metrics and status `healthy/degraded/manual_required/disabled`. A degraded source must not delete existing jobs and must surface a console warning.
- [ ] Run tests; expect PASS.
- [ ] Commit `feat: detect campus source changes`.

### Task 5: Implement adapters only from reproduced form failures

**Files:** Create `apps/extension/src/adapters/registry.ts`; when a reproduced failure requires it, create the matching folder under `apps/extension/src/adapters/{bytedance,kimi,meituan,pdd,baidu,kuaishou,xiaohongshu}` with a redacted fixture and contract test.

- [ ] Run Phase 2 generic engine against each approved fixture and record the exact unsupported controls.
- [ ] For every reproduced failure, write a failing adapter contract test. Do not create an adapter for a company whose fixture passes generically.
- [ ] Implement only the missing inspection/action hooks; delegate standard controls to the generic engine. Each adapter declares supported hostnames and fixture version.
- [ ] Run extension unit and e2e tests; expect generic behavior unchanged and adapter cases PASS.
- [ ] Commit each required adapter separately with the concrete control and company in the message, for example `fix: support searchable school select on bytedance applications`.

### Task 6: Conduct seven submission-free rehearsals

**Files:** Create `docs/qa/site-rehearsal-template.md` and `docs/qa/2026-07-21-{bytedance,kimi,meituan,pdd,baidu,kuaishou,xiaohongshu}.md`.

- [ ] Record job URL, browser/version, detected source/adapter, total fields, verified fills, review fields, blanks, upload result, and blockers.
- [ ] Run one real public job through discovery → package → fill → pre-submit review for every target company. User handles login/CAPTCHA; stop before submit.
- [ ] Calculate standard-field accuracy as `verified planned fields / attempted standard fields`; require at least 90% overall and zero high-risk guesses.
- [ ] Fix reproduced failures using tests before repeating a failed rehearsal.
- [ ] Commit `test: document seven site rehearsals` after all acceptance checks pass.

## Phase 3 acceptance

All seven companies support either automated public discovery or explicit manual capture with a recorded reason. Jobs normalize and deduplicate without silent deletion. Each site reaches pre-submit review in a real rehearsal, overall standard-field accuracy is at least 90%, and no test or implementation bypasses login, CAPTCHA, access policy, or final submission.
