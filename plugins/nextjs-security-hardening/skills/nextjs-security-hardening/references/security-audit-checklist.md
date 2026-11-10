# Pre-Deployment Security Audit — Grep + Checklist

This is the working reference for running the audit pattern described in
SKILL.md §12. Use it as the dispatch brief when calling `delegate_task`
with `tasks: [...]` for two parallel subagent audits.

---

## Subagent 1 — Project-internal audit (dispatch brief)

Replace placeholders in brackets. The subagent works inside the project
directory and returns a report; it does not modify code.

**Goal:** Find every security issue that would break production or leak
user data. Don't fix anything — only report findings with file:line.

**Repo context:**
- Next.js 14 App Router project at [PROJECT_DIR]
- `npm run build` is the build command (uses `next build`)
- `tsconfig.json` is at the root; `next build` does the typecheck
- `package.json` test scripts: [LIST THE TEST SCRIPTS]
- Customer-facing site at [DOMAIN], admin at [DOMAIN]/admin

**Recent work:** [BRIEFLY LIST RECENTLY BUILT FEATURES — this helps the
subagent know what surface area to focus on]

**Scope of audit — check ALL of these, in order:**

### 1. Build
Run `npx next build` and report errors/warnings. Build failures are
gates — a single TS error blocks a deploy.

### 2. TypeScript
Run `npx tsc --noEmit` and report type errors that `next build` didn't
catch. Sometimes tsc is more strict than the build.

### 3. Test suite
Run every test script in `scripts/`. Report pass/fail per script. Common
patterns:
- `npx tsx scripts/test-X.ts`
- `npx tsx scripts/test-X.test.ts`

### 4. Security scan
Read the code and report on:
- **API routes with missing auth or rate limiting**
  - `grep -rnE "export async function (GET|POST|PATCH|DELETE)" app/api/`
  - For each route, check: is there a `requireAdmin` / session check? Is
    there `applyRateLimit`?
- **Service role key exposure to the client**
  - `grep -rn "SUPABASE_SERVICE_ROLE_KEY\|SERVICE_KEY" src/app/`
  - Service role keys must never appear in client components
- **User input that goes into SQL or shell commands without sanitization**
  - `grep -rnE "\.raw\(|exec\(|spawn\(|query\(.*\+\s*\w+" app/api/`
- **CORS misconfiguration** — any route with custom `Access-Control-*` headers
- **File uploads that don't validate type/size**
  - `grep -rnE "formData\(\)|upload" app/api/`

### 5. PII enumeration holes (see SKILL.md §9)
- `grep -rnE "generate.*Id|nextval|increment|uuid|nanoid" lib/ app/`
- For each ID generator: sequential pad, counter, or monotonic? (Enums
  = vulnerable.)
- `grep -rnE "searchParams.get\('id'\)|params\.id" app/api/`
- For each ID-taking endpoint: does it require auth? Does the response
  contain PII fields (`customerName`, `customerEmail`, `customerPhone`,
  `shippingAddress`, `paymentId`, `trackingNumber`)?
- **The "?" mode trap:** a `?id=X` query is still a public endpoint if
  the route has no auth check.

### 6. Client-computed state (see SKILL.md §10)
- `grep -rnE "body\.(price|total|amount|discount|status|original_status|new_status|role|is_admin)" app/api/`
- For each match: is the value trusted, or re-derived server-side?
- Especially check: financial calculations, status transitions, role
  checks. Numbers and enums from body are the worst offenders.

### 7. Timing-unsafe compare (see SKILL.md §11)
- `grep -rnE "(apiKey|secret|signature|expected|provided|got)\s*(===|!==)\s*(\w+|\`)" app/api/`
- For each match: is the value a high-entropy secret? If yes, replace
  with `timingSafeEqual`.

### 8. Cron endpoints
- Every `app/api/cron/**/route.ts` should:
  - Read `CRON_SECRET` from env
  - Compare with `timingSafeEqual` (not `===`)
  - Reject with 401/503 on mismatch
  - Apply rate limiting (even crons can be hammered)

### 9. SSE / streaming endpoints
- Server-Sent Events (`EventSource`, `ReadableStream`) can be
  connection-flooded. Apply rate limiting to the GET that opens the
  stream. Set `Cache-Control: no-cache` and a hard cap on concurrent
  connections per IP.

### 10. Import paths
- Find any file that imports from a path with obviously wrong depth:
  `grep -rn "from '\.\./" app/ components/ lib/`
- Especially `.jsx` files in projects where `tsconfig.json` only
  includes `.ts`/`.tsx` — the `@/` alias doesn't resolve for them,
  so they must use relative paths. Wrong-depth imports will fail at
  build time.

### 11. SQL migration
- All `CREATE TABLE` statements should have RLS enabled
- All necessary indexes created
- No syntax errors
- Idempotent if possible (use `IF NOT EXISTS`)

### 12. Env vars
- Read the example environment file and cross-reference every `env.*`
  in `src/`. Any environment variable referenced in code but not documented is a
  deployment-time surprise.

### 13. Command-execution routes (see SKILL.md §13)
- `grep -rnE "execSync|exec\(|spawn\(|spawnSync|child_process|execFile" app/api/`
- `grep -rnE "shell:\s*true" app/api/`
- For each hit: what authenticates the caller BEFORE the command runs?
- A `startsWith` / fixed-command allowlist is NOT authentication. Report
  it as critical even when the route looks internal.
- Note what the server process identity grants beyond the command itself
  (credential files, SSH keys, cloud credentials) and whether stdout is returned.

**Report format:**
For each category, give a verdict: PASS / WARN / FAIL.
For each FAIL or WARN, give a 1-2 sentence description and the
file:line. If you find anything genuinely broken, list it at the top as
a "Critical findings" section.

Use file tools (`read_file`, `search_files`, `terminal`). Do NOT
modify code. Maximum tool budget: 50. Take your time.

---

## Subagent 2 — Cross-project / external audit (dispatch brief)

Working directory: a different project that the project-internal one
talks to via API. Examples: the command center for the health client, the health client for
the target platform, etc.

**Scope:**

1. **Build** — `npx next build` clean?

2. **Email templates** (if the project sends email) — read every file
   in `data/email-templates/`. Check:
   - All `{{...}}` placeholders match what the rendering function
     substitutes (no missing keys, no dangling unknown placeholders)
   - No XSS risks (user-controlled fields embedded in templates
     without escaping)
   - Brand assets render correctly (inlined SVG, no broken `<img>`)

3. **Send-due / scheduled-send endpoints** — check:
   - Bearer auth (CRON_SECRET)
   - Retry logic
   - State transitions on the queue table

4. **CSV export / bulk download endpoints** — check:
   - Auth (X-Admin-Key)
   - CSV escaping (commas, quotes, newlines in field values)
   - Memory usage — does it stream the CSV or build it all in memory?
     At 10k rows, in-memory build OOMs the server.

5. **Admin API endpoints** — check:
   - X-Admin-Key validation (use timingSafeEqual, not `===`)
   - Fail-closed behavior when key is missing
   - Read endpoints that should require the key but don't

6. **Public read endpoints** — any `/api/*` route that returns records
   with PII. Specifically check for ID-enumeration holes the same way
   Subagent 1 does, but from the cross-project angle (the the health client
   admin might call the command center's public endpoint to render data; if
   the command center's public endpoint leaks, the health client leaks it too).

7. **Storage path validation** — any endpoint that takes a file path or
   storage key from the client. Must validate the path doesn't escape
   the expected bucket.

**Report format:** same as Subagent 1.

---

## Dispatch syntax

```typescript
delegate_task(
  tasks: [
    {
      goal: "<subagent 1 goal + scope above>",
      context: "<working dir, repo info, recent work>",
      role: "leaf"
    },
    {
      goal: "<subagent 2 goal + scope above>",
      context: "<working dir, repo info, what API it serves>",
      role: "leaf"
    }
  ]
)
```

The batch returns one consolidated result. Don't read the live
transcript unless one of the subagents hangs — most audits complete
in 5-10 minutes.

---

## After the report comes back

The reports are self-reported, not verified fixes. Verify each
critical/high finding yourself before fixing:

1. Read the cited file:line
2. Reproduce the issue mentally (or with a curl)
3. Apply the fix
4. Re-run the build and tests

Don't trust subagent fixes — they often have edge cases the
subagent didn't see.
