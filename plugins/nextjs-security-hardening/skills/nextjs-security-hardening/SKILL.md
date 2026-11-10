---
name: nextjs-security-hardening
description: "Harden a Next.js app: AI bot protection, DevTools deterrence, rate limiting, security headers, and an audit of what is actually exposed."
version: 1.0.0
license: MIT
author: "the maintainers"
---
# Next.js Security Hardening

Comprehensive security layer for Next.js apps — bot protection, DevTools deterrence, rate limiting, audit logging, and a **pre-deployment security audit pattern** that catches 7+ critical classes of vulnerabilities in a single pass.

**Built for:** the target platform, the command center, and any future Next.js project the maintainer deploys.

---

## Stack of Layers (in order of deployment)

| Layer | What it does | File |
|-------|--------------|------|
| `robots.txt` | Tells AI crawlers to stay away | `public/robots.txt` |
| Security headers | Browser-enforced protections | `next.config.mjs` |
| Edge middleware | Blocks AI/bot UAs at the request level | `src/middleware.ts` |
| AI meta tags | HTML-level opt-out for specific crawlers | `src/app/layout.tsx` |
| DevTools blocker | Deterrent overlay if DevTools opens | `src/components/DevToolsBlocker.tsx` |
| Rate limiting | Prevents API abuse and enumeration | `src/lib/rateLimit.ts` + all route files |
| Audit logging | Records when DevTools blocker fires | `src/app/api/audit/devtools-log/route.ts` |
| Supabase RLS | Database-level access control | Verified in `supabase-schema.sql` |
| **Pre-deploy audit** | **Subagent-driven review for 11 vulnerability classes** | **See section 12** |

---

## Implementation Steps

### 1. robots.txt

Create `public/robots.txt` blocking all known AI/ML crawlers:

```
User-agent: GPTBot
Disallow: /

User-agent: ChatGPT-User
Disallow: /

User-agent: Claude-Web
Disallow: /

User-agent: Claude
Disallow: /

User-agent: anthropic-ai
Disallow: /

User-agent: Google-Extended
Disallow: /

User-agent: Bytespider
Disallow: /

User-agent: CCBot
Disallow: /

User-agent: Amazonbot
Disallow: /

User-agent: DeepSeek
Disallow: /

User-agent: Ai2bot
Disallow: /

User-agent: Diffbot
Disallow: /

User-agent: FacebookBot
Disallow: /

User-agent: GoogleOther
Disallow: /

User-agent: Applebot
Disallow: /

User-agent: DuckDuckBot
Disallow: /

User-agent: *
Disallow: /api/
Disallow: /admin/
Disallow: /dashboard/
Disallow: /partners/
```

### 2. Security Headers (next.config.mjs)

```javascript
const securityHeaders = [
  {
    key: 'X-Robots-Tag',
    value: 'noai, noindex, nofollow',
  },
  {
    key: 'Content-Security-Policy',
    value: [
      "default-src 'self'",
      "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://cdn.jsdelivr.net",
      "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
      "font-src 'self' https://fonts.gstatic.com",
      "img-src 'self' data: https: blob:",
      "media-src 'self' https: blob:",
      "connect-src 'self' https://*.supabase.co https://api.stripe.com https://*.railway.app",
      "frame-src 'self' https://js.stripe.com https://hooks.stripe.com",
      "frame-ancestors 'none'",
      "base-uri 'self'",
      "form-action 'self'",
      "upgrade-insecure-requests",
    ].join('; '),
  },
  {
    key: 'X-Frame-Options',
    value: 'DENY',
  },
  {
    key: 'X-Content-Type-Options',
    value: 'nosniff',
  },
  {
    key: 'Referrer-Policy',
    value: 'strict-origin-when-cross-origin',
  },
  {
    key: 'Permissions-Policy',
    value: 'camera=(), microphone=(), geolocation=(), interest-cohort=(), payment=()',
  },
];
```

Apply to `next.config.mjs`:
```javascript
async headers() {
  return [
    { source: '/:path*', headers: securityHeaders },
  ];
}
```

### 2b. Dev-only frame relaxation (Hermes / embedded preview) — that month

**Problem:** `X-Frame-Options: DENY` + CSP `frame-ancestors 'none'` are correct for production, but they blank Hermes Desktop's `open_preview` pane (iframe). Middleware often **re-sets** `X-Frame-Options: DENY` on every response — fixing only `next.config.mjs` is not enough.

**Rule:** relax framing **only when `NODE_ENV === 'development'`**. Production stays DENY / `frame-ancestors 'none'`.

```javascript
// next.config.mjs
// `env` is the environment object the framework hands you —
// Next.js exposes one, Vite exposes another. Read the value, not a secret.
const env = getEnvironment();
const isDev = env.NODE_ENV === 'development';
const frameOptions = isDev ? 'SAMEORIGIN' : 'DENY';
const frameAncestors = isDev
  ? "'self' http://localhost:* http://127.0.0.1:*"
  : "'none'";
// CSP: `frame-ancestors ${frameAncestors}`
// Omit upgrade-insecure-requests in dev
```

```typescript
// middleware.ts (must match config — both fire)
response.headers.set(
  'X-Frame-Options',
  env.NODE_ENV === 'development' ? 'SAMEORIGIN' : 'DENY'
);
```

**Pitfalls:** Config change needs dev-server restart. If Hermes origin still won't embed under SAMEORIGIN, fall back to Safari (`open -a Safari`) — never weaken production headers for the agent pane. Worked example: `the org-os` → `[removed: internal file]`.

### 3. Edge Middleware (src/middleware.ts)

Blocks AI/bot UAs at the edge before the app even loads. Replace the empty default middleware.

**Key patterns:**
- Maintain a `BLOCKED_UA_PATTERNS` array of regex patterns
- Sensitive paths (`/api/*`, `/admin/*`, etc.) are fully blocked for any matching UA
- Public paths return 403 with a plain text message — no HTML, no React, no JS loaded
- Use `request.headers.get('user-agent')` — NOT `request.nextUrl.userAgent` (doesn't exist)

**Critical bug:** `const { pathname, userAgent } = request.nextUrl` throws TS2339 — `userAgent` is not on `NextURL`. Fix:
```typescript
const { pathname } = request.nextUrl;
const userAgent = request.headers.get('user-agent') ?? '';
```

**Pitfall — exempt webhook endpoints from the UA block, not just from rate limiting.** The `BLOCKED_UA_PATTERNS` list (which includes `/curl/i`, `/python-requests/i`, `/axios/i`, etc.) will 403 every inbound POST from a third-party service whose User-Agent matches one of those patterns. Resend, Stripe, GitHub, and many other webhook senders send a UA like `Resend/1.0` or `Stripe/1.0` which can match depending on your pattern set. **Always maintain a `UA_BLOCK_EXEMPT_PATHS` array** (parallel to the rate-limit-skip list) and check the pathname against it BEFORE applying the UA block. Convention: any path that starts with `/api/<vendor>/webhook`, `/api/email/webhook`, `/hook/<vendor>`, or `/api/push/*` should be in the exempt list. Real case (the target platform corporate email, that month): the inbound webhook at `/api/email/webhook/inbound` was 403-blocked by the AI bot UA pattern until the path was added to `UA_BLOCK_EXEMPT_PATHS` — emails were being received by Resend, stored in the Receiving API, but never reached the app, and the user saw "no emails coming through." Symptom: every webhook receiver returns 403 from the edge middleware before the route handler runs; Resend/Stripe/GitHub dashboard shows the event was "delivered" but the app never sees it. Diagnostic: `curl -H "User-Agent: <service-default>" <webhook-url>` — if you get `{"error":"Access denied."}` with status 403, the middleware is blocking. The fix is one line in `UA_BLOCK_EXEMPT_PATHS`.

### 4. AI Meta Tags (src/app/layout.tsx)

Add per-route meta tags alongside the Next.js `metadata` export:
```typescript
const AI_BLOCK_TAGS = [
  { name: "robots", content: "noai, noimageai, noindex, nofollow, nosnippet, noarchive, notranslate" },
  { name: "googlebot", content: "noindex, nofollow" },
  { name: "anthropic-ai", content: "noindex" },
  { name: "GPTBot", content: "noindex" },
  { name: "Bytespider", content: "noindex" },
  { name: "CCBot", content: "noindex" },
  { name: "omgilabot", content: "noindex" },
];
```

Map these into `<head>` via a loop in the layout body.

### 5. Rate Limiting (src/lib/rateLimit.ts)

In-memory rate limiter using a `Map<string, { count: number; resetAt: number }>` pattern.

**Two tiers:**
- **Strict (10 req/min):** auth, signup, chat, checkout, trading credentials, partner register
- **Standard (30 req/min):** admin, partners, feed, email, leaderboard, trading dashboard

Apply to every route using `applyRateLimit(request, { limit: N, windowMs: 60_000 })`. Import from `@/lib/rateLimit`. Check the return — if not null, return it immediately.

**Pitfall:** Many routes were missing the `NextRequest` import. Always check that both `NextRequest` and `NextResponse` are imported from `'next/server'`.

**Skip rate limiting on:** webhook routes (`/api/webhooks/*`) — they have their own signature/token validation.

**Apply rate limiting to public-but-expensive endpoints:** SSE streams (`/api/chat/stream`), file uploads (`/api/*/upload`), and any endpoint a bad actor could spam to exhaust resources. Reasonable defaults: 30/min for SSE, 10/5min for uploads.

### 6. DevTools Blocker Component

A `"use client"` component that detects DevTools opening and shows a full-screen overlay.

**Detection methods:**
- Dimension diff (outerWidth - innerWidth > 160px threshold) — fires every 500ms via `setInterval`
- `window.addEventListener('resize')` — immediate detection for docked DevTools
- `window.addEventListener('orientationchange')` — mobile DevTools detection

**Deduplication:** Use a `useRef(false)` to only log once per DevTools-open session, not every cycle.

**Limitations:** Deterrent only — can be bypassed with JS disabled or browser flags. Stop casual inspection, not determined attackers.

### ⚠️ CRITICAL PITFALL — DevToolsBlocker False Positives on macOS Browsers (Jun 26 2026)

**Symptom:** The dimension-diff detector (`outerWidth - innerWidth > 160px`) fires FALSE POSITIVES on macOS Safari/Chrome/Zen/Arc when:
- The browser window has a sidebar (Bookmarks, Reading List, etc.) open
- The user has Dock previews visible
- The window is in certain split-screen / fullscreen states
- The browser has extension panels (React DevTools, Redux DevTools) docked

**User impact:** Full-screen black "Access Denied" overlay appears over the entire site for the legitimate owner/admin trying to preview the app. User reported: "this is showing up current on local host... but right now we are in preview mode so allow me to view the page please." The blocker literally blocked the site owner from viewing their own site.

**Why this is worse than no blocker:** A deterrent that fires on the people it's supposed to protect is actively harmful. The owner cannot review, edit, or test their own pages because the moment their browser has any sidebar open, the site becomes inaccessible.

**Fix options (pick one):**
1. **Remove the component from layout entirely** — log a comment explaining why. Use server-side detection at the edge middleware layer (UA strings) + RLS instead. This is the recommended default for owner-operated sites.
2. **Gate it behind an environment variable** — `env.NEXT_PUBLIC_ENABLE_DEVTOOLS_BLOCKER !== 'false'` so the owner can disable it locally without removing the file.
3. **Replace with a non-blocking overlay** — small toast at bottom-right saying "Inspection detected" instead of full-page block. Allows viewing + flags activity.

**If keeping the dimension detector:** Add a developer override:
```typescript
// At the top of detectByDimension()
if (window.location.search.includes('disable-devtools-blocker')) return;
// Then: ?disable-devtools-blocker in URL to bypass
```

**For site owners debugging their own site:** When the blocker fires unexpectedly, the FIRST diagnostic check is `window.outerWidth - window.innerWidth` in the browser console. If it's > 160px with DevTools CLOSED, the blocker has a false positive. Browser-side docked panels (Bookmarks sidebar, etc.) are the usual culprit.

**Recognition pattern in user messages:** If the user says "this looks like a preview mode" or "I can't see my own page" or "the page is showing a black overlay" right after you turned on the DevToolsBlocker, it's the false positive — not a real "preview mode" the user opted into.

### 7. Audit Logging Route

`POST /api/audit/devtools-log` — server-side endpoint that:
- Reads real IP from `x-forwarded-for` (first comma-separated value), `cf-connecting-ip`, `x-real-ip`, or `x-client-ip`
- Appends entry to a local JSON log file (`the org-devtools-audit.json`) capped at 5,000 entries
- Also emits structured JSON to Railway logs: `{"type":"devtools_blocker_fired", ...}`

`GET /api/audit/devtools-log` — server-side read of the log file (not exposed to clients).

**Do NOT add this route to rate limiting** — it's server-side only and not accessible to external clients.

### 8. Supabase RLS Verification

Run this in the Supabase SQL editor to audit all tables:
```sql
SELECT tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'public'
AND rowsecurity = false;
```

Any table with `rowsecurity = false` that shouldn't be public is a gap. All affiliate, trading, and user-data tables must have RLS enabled.

### 9. Resource Enumeration via Sequential IDs (PII Leak Vector)

**The vulnerability class.** Any endpoint that returns a single record by ID and IDs are sequential, guessable, or returned in any unrelated response is a PII enumeration hole. Worst-case impact: an attacker can dump every customer's name, email, phone, and shipping address with `curl` and a counter.

**Recognition pattern (audit-time checklist):**
1. **Find the ID generator.** `grep -rnE "generate.*Id|nextval|increment|uuid|nanoid" lib/ app/`. If the generator pads a counter (`ord_vt_001`, `ord_vt_002`, ...) or exposes a monotonic number, IDs are enumerable.
2. **Find every endpoint that takes an ID.** `grep -rnE "/\[id\]|searchParams.get\('id'\)|params\.id" app/api/`. Any endpoint that returns a record by ID is a candidate.
3. **Check auth on each ID-taking endpoint.** If `requireAdminKey` (or session check) is missing, the endpoint is public.
4. **Check whether the response contains PII.** `customerName`, `customerEmail`, `customerPhone`, `shippingAddress`, `paymentId`, `trackingNumber` — these are the fields that make an enumeration attack worth running.

**The "?" mode trap.** Routes that *appear* to need a query string (`?id=X`) are still public endpoints. The check is whether the route requires auth before it processes the ID, not whether the ID is in the path.

**Real case (the health client, that month):** `app/api/client/orders/route.ts` had no `requireAdminKey` guard. It supported `?id=ord_vt_001` and returned the full order including `customerEmail`, `customerPhone`, and `shippingAddress`. The ID generator was `generateOrderId` in `lib/commerce.ts` — a `padStart(3, '0')` counter. The admin equivalent (`/api/client/admin/orders/[id]/route.ts`) *did* require `X-Admin-Key`. So the same data was available with and without auth, and the IDs were sequential. **FAIL severity — fix by either gating the public endpoint with `requireAdminKey`, removing the `?id=` mode, or issuing per-order signed lookup tokens.**

**Fixes (pick one):**
- **Gate behind admin key.** Add `requireAdminKey` at the top of the public read endpoint. Cheapest fix when no real "customer looks up their own order" UX exists.
- **Signed lookup token.** When the customer-facing UX is real (thank-you page, GitHub-issue-style status link), generate a per-order random token at write time, store it on the record, and only accept that token in the public URL. Reject token mismatches with 404. Never use the order ID as the token.
- **UUID IDs instead of sequential.** Swallows the problem but doesn't remove it — UUIDs are still enumerable if the endpoint leaks them in batch responses. Always pair with auth.
- **Rate limit per IP.** Defense in depth, not a fix. Apply alongside the gating change.

**Pitfall:** batch endpoints (orders list, admin search) leak IDs. Even if the single-record endpoint is gated, a public list endpoint that returns `{ id, customerName, customerEmail }` rows lets an attacker harvest IDs and then query the gated endpoint. Audit the **shape** of every public response, not just whether an endpoint exists.

### 10. Client-Computed Financial / State Values

**The vulnerability class.** Any endpoint that accepts a computed value (price, discount, total, status transition, role flag) from the request body instead of re-deriving it from server-side state lets a client manipulate the outcome by submitting fake values.

**Recognition pattern (audit-time checklist):**
1. `grep -rnE "body\.(price|total|amount|discount|status|original_status|new_status|role|is_admin)" app/api/`
2. For each match, ask: "Could a malicious caller submit a value here that's different from the server's correct value?" If yes, server-side re-derivation is required.
3. Check the field's type: is it a number, enum string, or boolean? Numbers and enums are the worst — clients can submit any value.

**Real cases (the health client, that month audit):**
- `/api/upgrade/checkout` trusted `body.price_difference` from the client. A bad actor could submit `price_difference: 0` and get a free upgrade. Fix: re-derive the price from `from_level`, `to_level`, and any discount flag using a server-side `lib/upgrade-pricing.ts` module. **The client-supplied price is ignored.**
- `/api/admin/support` PATCH read `body.original_status` to decide whether to email the customer about a status change. A malicious admin (or anyone with a valid session) could spoof the previous status to suppress customer notifications. Fix: read the current status from the DB **before** applying the update, then derive the email trigger from the before→after diff in the response.

**Rule:** any value that drives a state transition, a financial calculation, or a notification should be re-derived server-side. Client values are inputs to a calculation, never the calculation itself.

### 11. Timing-Unsafe Secret Compare

**The vulnerability class.** Using `===` or `!==` to compare an incoming secret (API key, cron secret, webhook signature) against a server-side expected value leaks the expected value one byte at a time via timing differences. Modern JS engines short-circuit on the first non-matching byte, so an attacker can measure response times and brute-force the secret byte-by-byte.

**Recognition pattern (audit-time checklist):**
1. `grep -rnE "(apiKey|secret|signature|expected|provided|got)\s*(===|!==)\s*(\w+|\`)" app/api/`
2. For each match, check if the value being compared is a high-entropy secret (API key, HMAC, token). If yes, the `===` is a vulnerability.
3. Replace with Node's `crypto.timingSafeEqual` — wrapped in a length check first (because `timingSafeEqual` throws on unequal lengths, which would itself leak the expected length via timing). Use `Buffer.from` to convert strings to bytes. Cast to `Uint8Array` if the type system complains (Next.js TypeScript strict).

**Reference implementation:**
```typescript
import { createHmac, timingSafeEqual } from 'crypto'

function constantTimeEqual(provided: string, expected: string): boolean {
  const a = new Uint8Array(Buffer.from(provided))
  const b = new Uint8Array(Buffer.from(expected))
  if (a.length !== b.length) return false
  return timingSafeEqual(a, b)
}
```

**Real cases (Jul 2026 audit):** cron endpoint used `!==` for `CRON_SECRET`; Square webhook export endpoint used `!==` for `X-Admin-Key`. Both fixed.

**Severity:** low-to-medium in practice (32+ random bytes take geological time to brute-force even with timing), but trivially fixable and the right thing to do.

### 12. The "Subagent Audit" Pattern — Pre-Deployment Security Review

For a thorough pre-deployment or post-deploy security review, dispatch **two parallel subagent** audits with these scopes:

**Subagent 1 — Project-internal audit.** Working directory: the Next.js project root. Scans all code paths for the 11 vulnerability classes above (PII enumeration, ID leakage, timing-unsafe compare, missing rate limits, missing auth, client-computed state, etc.). Builds, type-checks, and runs the test suite. Reports prioritized findings with file:line references.

**Subagent 2 — Cross-project / external audit.** Working directory: a different project that talks to the first via API (e.g., the command center proxies for the health client). Scans email pipelines, webhook handlers, and any third-party API integrations (Square, Resend, the broker, Stripe). Reports findings independently.

**Run them in parallel via `delegate_task` with `tasks: [...]` syntax** — the consolidated results come back in one round trip. You don't have to wait sequentially.

**Why two subagents:** A single agent reviewing the same codebase tends to miss classes of bugs that the code "obviously" handles. Two agents with different contexts catch different things. The internal agent catches PII and ID-leak issues in the main app; the external agent catches issues in the supporting services (email templates, webhooks, third-party integrations) that the main agent doesn't have visibility into.

**Important: do NOT fix anything in the subagent call.** Subagent returns are self-reports, not verified fixes. Read the report, prioritize, fix the high-severity items yourself (or in a follow-up pass), then verify with a real test or build.

**Recognition pattern in user messages:** if the user says "we're good, do an audit" or "before we go live, make sure everything's working," dispatch this pattern immediately rather than trying to audit the entire codebase yourself in one pass.

**Expected yield:** a single 2-subagent audit pass on a Next.js app of ~100-200 routes typically finds 5-10 real issues ranging from critical (PII enumeration, missing rate limits) to low (typos in customer-visible strings, missing touch targets). Don't skip this step before any "go live" or "ready for customers" milestone.

### 13. Unauthenticated Command-Execution Routes

**The vulnerability class.** A Next.js API route that spawns a shell and is
not gated by a real secret is remote code execution, not an integration. The
server process's own privileges become the caller's privileges.

**Recognition pattern (audit-time checklist):**
1. `grep -rnE "execSync|exec\(|spawn\(|spawnSync|child_process|execFile" app/api/` and
   `grep -rnE "shell:\s*true" app/api/` — every hit is a candidate.
2. For each hit, find the auth check that runs **before** the command.
3. A string-prefix or fixed-command allowlist is **not** authentication — it
   constrains what a caller may type, not who they are. A prefix whose file
   does not exist still satisfies the check and the shell continues past it.
4. The correct order is **secret first, then command shape**: verify a shared
   secret with `crypto.timingSafeEqual`, *then* validate the command.

**Real case (the command center, Nov 2026):** `/api/system/swarm-exec` accepted a
caller-supplied shell string and ran it as the server user. Its only gate was
a `startsWith` check against a hardcoded prefix naming a workspace script.
The referenced script did not exist on the target machine, so the interpreter
printed an error to stderr while the shell's `;` carried on into the rest of
the caller's command — the prefix was satisfied and the allowlist was
inert. The route had no secret at all, and the app was reachable on a
tailnet address, so any device on that network got shell execution on the
host.

**Fixes (pick one):**
- **Named-operation endpoint.** Map the route to a fixed argv; the caller
  sends no command string. Best when the use case is one narrow job.
- **Secret + allowlist.** Verify a shared secret first, then constrain the
  command. Both, in that order.
- **Unprivileged execution.** Run the command as a restricted user with a
  fixed `PATH` and no access to the app's env file or SSH keys.

**Severity:** critical, always. A "private" tailnet address is reachable by
every device on that network — a private network is a topology, not a
credential.

Full audit procedure: `references/command-execution-route-audit.md`.

---

## Deployment Verification Checklist

After every deploy to Railway, verify:

```bash
# 1. Check Railway logs for startup
railway logs

# 2. Verify robots.txt (use GET, not HEAD — HEAD returns 404 on some Next.js static routes)
curl -s "https://<domain>/robots.txt"

# 3. Verify security headers
curl -sI "https://<domain>" | grep -iE "x-frame|x-content|x-robots|content-security|referrer|permissions"

# 4. Test rate limiting (should get 429 after N+1 requests)
for i in $(seq 1 12); do
  curl -s -o /dev/null -w "%{http_code}\n" "https://<domain>/api/<route>"
done
```

---

## Key Files Summary

| File | Purpose |
|------|---------|
| `public/robots.txt` | AI crawler block list |
| `next.config.mjs` | Security headers |
| `src/middleware.ts` | Edge UA blocking |
| `src/app/layout.tsx` | AI meta tags |
| `src/lib/rateLimit.ts` | In-memory rate limiter |
| `src/components/DevToolsBlocker.tsx` | DevTools deterrent |
| `src/app/api/audit/devtools-log/route.ts` | Audit logger |
| `supabase-schema.sql` | RLS audit |
| `references/security-audit-checklist.md` | Audit grep + checklist |
| `references/command-execution-route-audit.md` | Auditing routes that spawn a shell — auth vs. command-shape allowlists, the prefix trap, and tailnet exposure |
| `references/mobile-pass-a-css.md` | Mobile touch-target + safe-area CSS snippets |

---

## Related Skills

- **`deploy`** — Railway deployment and verification patterns
- **`supabase-projects`** — RLS audit queries
- **`ecommerce-payment-readiness-audit`** — E-commerce-specific pre-launch checklist
