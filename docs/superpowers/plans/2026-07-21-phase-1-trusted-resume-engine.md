# Phase 1 Trusted Resume Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a local application that imports resume facts, requires user verification, parses JDs, produces evidence-backed matches, and exports traceable application packages.

**Architecture:** Use a pnpm TypeScript monorepo with a Fastify local API, SQLite persistence, shared Zod contracts, and a React/Vite console. All model output is parsed as structured JSON and rejected unless every generated claim cites confirmed facts.

**Tech Stack:** Node.js 22, TypeScript, pnpm workspaces, Fastify, better-sqlite3, Drizzle ORM, Zod, Vitest, React, Vite, TanStack Query, docx, Playwright PDF, OpenAI-compatible API.

---

## File map

- `package.json`, `pnpm-workspace.yaml`, `tsconfig.base.json`: monorepo commands and compiler policy.
- `packages/contracts/src/*`: shared domain schemas; contains no I/O.
- `apps/core/src/db/*`: schema, migrations, and repositories.
- `apps/core/src/import/*`: DOCX/PDF extraction and fact candidates.
- `apps/core/src/ai/*`: redaction, provider interface, structured responses, claim validation.
- `apps/core/src/applications/*`: matching and package generation.
- `apps/core/src/server.ts`: local-only HTTP composition root.
- `apps/console/src/*`: fact review, JD intake, match, and package UI.

### Task 1: Bootstrap the typed monorepo

**Files:**
- Create: `package.json`
- Create: `pnpm-workspace.yaml`
- Create: `tsconfig.base.json`
- Create: `packages/contracts/package.json`
- Create: `packages/contracts/src/index.ts`
- Test: `packages/contracts/src/index.test.ts`

- [ ] **Step 1: Write the failing schema test**

```ts
import { describe, expect, it } from "vitest";
import { ProfileFactSchema } from "./index";

describe("ProfileFactSchema", () => {
  it("rejects an unconfirmed fact used for generation", () => {
    const result = ProfileFactSchema.safeParse({
      id: "fact_1", category: "skill", value: "TypeScript",
      normalizedValue: "typescript", verificationStatus: "unknown",
      sensitivityLevel: "professional", updatedAt: "2026-07-21T00:00:00.000Z"
    });
    expect(result.success).toBe(true);
    if (result.success) expect(result.data.verificationStatus).toBe("unknown");
  });
});
```

- [ ] **Step 2: Run the test and verify bootstrap failure**

Run: `pnpm vitest packages/contracts/src/index.test.ts --run`  
Expected: FAIL because workspace files and schema do not exist.

- [ ] **Step 3: Add workspace configuration and domain schemas**

```ts
// packages/contracts/src/index.ts
import { z } from "zod";

export const ProfileFactSchema = z.object({
  id: z.string().min(1), category: z.string().min(1), value: z.string(),
  normalizedValue: z.string(), sourceDocument: z.string().optional(),
  sourceLocation: z.string().optional(),
  verificationStatus: z.enum(["unknown", "confirmed", "rejected"]),
  sensitivityLevel: z.enum(["public", "professional", "personal", "restricted"]),
  updatedAt: z.string().datetime()
});
export type ProfileFact = z.infer<typeof ProfileFactSchema>;
export const ConfirmedFactSchema = ProfileFactSchema.extend({ verificationStatus: z.literal("confirmed") });
```

Root scripts must include `typecheck`, `test`, `lint`, and `dev`; package manager must be pinned via `packageManager`.

- [ ] **Step 4: Install dependencies and run checks**

Run: `pnpm install && pnpm test && pnpm typecheck`  
Expected: PASS; one contract test passes.

- [ ] **Step 5: Commit**

```bash
git add package.json pnpm-workspace.yaml tsconfig.base.json packages
git commit -m "chore: bootstrap typed workspace"
```

### Task 2: Persist confirmed facts and audit changes

**Files:**
- Create: `apps/core/src/db/client.ts`
- Create: `apps/core/src/db/schema.ts`
- Create: `apps/core/src/facts/fact-repository.ts`
- Test: `apps/core/src/facts/fact-repository.test.ts`

- [ ] **Step 1: Write repository tests**

```ts
it("keeps an audit row when a candidate is confirmed", () => {
  const repo = makeTestFactRepository();
  repo.upsert(candidateFact);
  repo.setVerification(candidateFact.id, "confirmed", "user");
  expect(repo.getConfirmed()).toHaveLength(1);
  expect(repo.getAudit(candidateFact.id).at(-1)?.actor).toBe("user");
});
```

- [ ] **Step 2: Verify failure**

Run: `pnpm --filter @auto-find-job/core test -- fact-repository.test.ts`  
Expected: FAIL because repository is missing.

- [ ] **Step 3: Implement SQLite schema and repository**

Create `profile_facts` and append-only `fact_audit` tables. `setVerification` must use one transaction and reject missing fact IDs. Store timestamps as ISO UTC strings; never overwrite audit rows.

```ts
setVerification(id: string, status: VerificationStatus, actor: "user" | "system") {
  return this.db.transaction(() => {
    const before = this.get(id);
    if (!before) throw new Error(`FACT_NOT_FOUND:${id}`);
    this.updateStatus.run(status, new Date().toISOString(), id);
    this.insertAudit.run(crypto.randomUUID(), id, JSON.stringify(before), status, actor);
  })();
}
```

- [ ] **Step 4: Run repository and type tests**

Run: `pnpm --filter @auto-find-job/core test && pnpm typecheck`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/core
git commit -m "feat: persist verified profile facts"
```

### Task 3: Import DOCX and PDF as unconfirmed candidates

**Files:**
- Create: `apps/core/src/import/extract-text.ts`
- Create: `apps/core/src/import/parse-fact-candidates.ts`
- Create: `apps/core/src/import/import-service.ts`
- Test: `apps/core/src/import/import-service.test.ts`
- Test fixture: `apps/core/test/fixtures/resume.docx`

- [ ] **Step 1: Write tests for import safety**

```ts
it("never marks imported facts confirmed", async () => {
  const facts = await service.importResume(fixture("resume.docx"));
  expect(facts.length).toBeGreaterThan(0);
  expect(facts.every(f => f.verificationStatus === "unknown")).toBe(true);
  expect(facts.every(f => f.sourceDocument.endsWith("resume.docx"))).toBe(true);
});
```

- [ ] **Step 2: Verify failure**

Run: `pnpm --filter @auto-find-job/core test -- import-service.test.ts`  
Expected: FAIL because import service is missing.

- [ ] **Step 3: Implement bounded extraction**

Use `mammoth` for DOCX and `pdfjs-dist` for PDF. Reject unsupported MIME types and files over 10 MiB. Produce candidates with source document, page/section where available, `unknown` verification, and sensitivity classification. Do not call an LLM during raw extraction.

- [ ] **Step 4: Run tests**

Run: `pnpm --filter @auto-find-job/core test -- import-service.test.ts`  
Expected: PASS for DOCX, PDF, invalid MIME, and size limit cases.

- [ ] **Step 5: Commit**

```bash
git add apps/core/src/import apps/core/test/fixtures
git commit -m "feat: import resume fact candidates"
```

### Task 4: Add privacy-bounded AI matching and claim validation

**Files:**
- Create: `apps/core/src/ai/provider.ts`
- Create: `apps/core/src/ai/redact.ts`
- Create: `apps/core/src/ai/schemas.ts`
- Create: `apps/core/src/ai/claim-validator.ts`
- Create: `apps/core/src/applications/match-service.ts`
- Test: `apps/core/src/ai/claim-validator.test.ts`
- Test: `apps/core/src/applications/match-service.test.ts`

- [ ] **Step 1: Write adversarial claim tests**

```ts
it.each([
  { text: "提升转化率 30%", ids: [] },
  { text: "熟练使用 Rust", ids: ["fact_typescript"] }
])("rejects unsupported claim %#", ({ text, ids }) => {
  expect(() => validateClaim({ text, sourceFactIds: ids }, confirmedFacts)).toThrow();
});
```

- [ ] **Step 2: Verify failure**

Run: `pnpm --filter @auto-find-job/core test -- claim-validator.test.ts`  
Expected: FAIL because validator is missing.

- [ ] **Step 3: Implement provider boundary and validation**

Define `AiProvider.generateStructured<T>(schema, messages): Promise<T>`. Redaction must remove personal/restricted facts before serialization. Validate that every claim cites existing confirmed facts and that numbers, dates, organizations, and skills in the claim are present in cited normalized values. Store model and prompt versions with results.

- [ ] **Step 4: Run adversarial and matching tests**

Run: `pnpm --filter @auto-find-job/core test -- claim-validator.test.ts match-service.test.ts`  
Expected: PASS; unsupported claims fail closed.

- [ ] **Step 5: Commit**

```bash
git add apps/core/src/ai apps/core/src/applications
git commit -m "feat: add evidence-backed resume generation"
```

### Task 5: Generate versioned DOCX/PDF application packages

**Files:**
- Create: `apps/core/src/applications/package-service.ts`
- Create: `apps/core/src/documents/docx-renderer.ts`
- Create: `apps/core/src/documents/pdf-renderer.ts`
- Test: `apps/core/src/applications/package-service.test.ts`

- [ ] **Step 1: Write package manifest test**

```ts
it("writes a complete traceable manifest", async () => {
  const output = await service.generate(input);
  expect(output.files).toEqual(expect.arrayContaining(["resume.docx", "resume.pdf", "manifest.json"]));
  expect(output.manifest.claims.every(c => c.sourceFactIds.length > 0)).toBe(true);
});
```

- [ ] **Step 2: Verify failure**

Run: `pnpm --filter @auto-find-job/core test -- package-service.test.ts`  
Expected: FAIL.

- [ ] **Step 3: Implement deterministic renderers**

Render DOCX with `docx`. Render the same semantic resume model to print HTML and PDF via Playwright. Use `company-role/application-id/version` directories and atomic rename from a temporary directory. Manifest includes hashes of JD snapshot and every output file.

- [ ] **Step 4: Run package tests**

Run: `pnpm --filter @auto-find-job/core test -- package-service.test.ts`  
Expected: PASS and no partial directory after injected renderer failure.

- [ ] **Step 5: Commit**

```bash
git add apps/core/src/applications apps/core/src/documents
git commit -m "feat: generate versioned application packages"
```

### Task 6: Expose the local API and review console

**Files:**
- Create: `apps/core/src/server.ts`
- Create: `apps/core/src/routes/facts.ts`
- Create: `apps/core/src/routes/applications.ts`
- Create: `apps/console/src/App.tsx`
- Create: `apps/console/src/features/facts/FactReview.tsx`
- Create: `apps/console/src/features/applications/ApplicationBuilder.tsx`
- Test: `apps/core/src/server.test.ts`
- Test: `apps/console/src/features/facts/FactReview.test.tsx`

- [ ] **Step 1: Write API and UI tests**

```ts
it("binds only to loopback and rejects missing session token", async () => {
  const app = buildServer({ sessionToken: "test-token" });
  expect((await app.inject({ method: "GET", url: "/api/facts" })).statusCode).toBe(401);
});
```

- [ ] **Step 2: Verify failure**

Run: `pnpm test`  
Expected: FAIL because server and console are missing.

- [ ] **Step 3: Implement routes and review screens**

Bind to `127.0.0.1`, require a per-install token, validate all bodies with shared Zod schemas, and cap uploads at 10 MiB. UI must show source location and require explicit confirm/reject actions; generation remains disabled while referenced facts are unconfirmed.

- [ ] **Step 4: Run full phase checks**

Run: `pnpm test && pnpm typecheck && pnpm lint && pnpm --filter @auto-find-job/console build`  
Expected: all commands PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/core apps/console packages/contracts
git commit -m "feat: deliver trusted resume workflow"
```

## Phase 1 acceptance

Run `pnpm test && pnpm typecheck && pnpm lint`. Import both a DOCX and PDF, confirm facts, paste a JD, generate a package, and verify each claim in `manifest.json` cites confirmed facts. Inspect model request audit and verify personal/restricted fields are absent.

