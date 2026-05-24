# Curriculum: per-concept courses, videos, and exercises
Companion to `ROADMAP.md`. Generated 2026-05-22 via a 4-researcher / 2-reviewer / 2-validator agent team, then synthesised by hand.

`ROADMAP.md` gives you the phase map and the concept inventory. This file goes one layer deeper. For every concept worth your time, it lists the single best resource and a concrete exercise that triggers the failure mode before showing you the fix.

---

## How this file is structured (read this once)

- **Exercises are failure-driven.** Every exercise that survived review forces you to *cause* the bug first. Drift detection, a JWE-not-JWS decode, an action-ID curl bypass, a prepared-statement error, an RSC serialization break: in each case you see the failure before the fix. If you complete an exercise without hitting the red error first, you skipped a step.
- **Time budgets are honest.** They include setup. RS256+JWKS is not 60 minutes if you've never used `jose`.
- **Senior-skip lists are aggressive.** Anything a senior .NET/React dev finishes in 5 minutes ("install fnm", "rename `NEXTAUTH_SECRET`") was killed. ROADMAP's "what to SKIP" applies, and this file goes further.
- **Resources are opinionated.** If only the official docs are best-in-class, that's what's listed. No padding.

---

## Prerequisites: set these up ONCE before starting Phase 1

The realism validator caught that several mid-curriculum exercises silently assume infrastructure. Get this done in one batch so it stops being a hidden tax.

| Prereq | Why | One-time cost |
|---|---|---|
| Node 24 (via `fnm`) + corepack + pnpm 11 | Everything | 15 min |
| Local Postgres (Docker or native) with a named DB + DDL user | Phase 2 onward | 20 min |
| Docker daemon running, `docker compose` available | PgBouncer exercise (#24) | already installed |
| A `docker-compose.yml` for PgBouncer (write it once and reuse) | PgBouncer exercise | 30 min |
| GitHub OAuth App registered with `http://localhost:3000/api/auth/callback/github` | Auth.js exercises (#38+) | 10 min |
| `autocannon` and `openssl` available in PATH | Pool + JWKS exercises | 5 min |
| React DevTools browser extension | RSC verification | 2 min |
| Phase 1 Hono server (port 3001), kept running through Phase 3+ | Next.js cross-service exercises | reused |

If you skip this, Phase 5's "60-minute" JWKS exercise becomes a 4-hour rabbit hole.

---

## Critical 2026 corrections (caught by review; do not propagate the originals)

The research agents made three factual errors worth flagging up front:

1. The Hono JWK middleware CVE is `CVE-2026-22818` (advisory `GHSA-3vhc-576x-3qv4`), fixed in Hono 4.11.4+. The originally cited "CVE-2026-27804" is a Parse Server vulnerability in a completely different package. If you audit your Hono version, search for `GHSA-3vhc-576x-3qv4`.
2. `@auth/core/jwt` `decode()` takes `{ token, secret, salt }`, not `{ token, secret }`. `salt` is required and used with the secret to derive the decryption key. Omitting it will fail. See [authjs.dev/reference/core/jwt](https://authjs.dev/reference/core/jwt).
3. The `revalidateTag(tag)` single-argument form is deprecated with a TS warning, not a compile error. It still works in Next.js 16.2.6. Use `revalidateTag(tag, profile)` everywhere new, and don't expect the compiler to force the fix.

Several URLs the research agents cited are also dead or unreliable:
- `practica.dev/blog/...` renders blank. Treat it as broken.
- `x.com/yusukebe/status/...` is login-walled. Use [blog.yusu.ke/hono-rpc/](https://blog.yusu.ke/hono-rpc/) instead.
- `answeroverflow.com/m/...` 403s.
- `vercel.com/academy/...` shows "last updated today", which is an auto-timestamp rather than editorial currency. Don't trust it as proof of freshness.

---

# Phase 0: Toolchain (~2 hours, not really a "phase")

Reviewer A killed Phase 0 as exercise material. "Install fnm, verify `--version` prints" is not learning. Treat the whole phase as setup, done once, while you watch a video on something else.

- **Node 24 LTS + pnpm 11 + Biome v2 + Vitest.** Install in one sitting. Node 24 is LTS as of Oct 2025; pnpm 11 via `corepack enable && corepack prepare pnpm@latest-11 --activate`. Biome v2 (released [June 17, 2025](https://biomejs.dev/blog/biome-v2/)) is the opinionated pick over ESLint flat config for a greenfield Hono service.
- **Strict tsconfig.** This is the one Phase 0 exercise that survived. **Exercise (45 min):** create a `tsconfig.json` with `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `verbatimModuleSyntax`, and `isolatedModules`. Write three snippets that each compile under plain `"strict": true` but fail under one of these flags: (a) array index access without undefined check, (b) assigning `undefined` to a `field?:` property, (c) a regular `import` where `import type` is required. Reference: [Matt Pocock's TSConfig cheat sheet](https://www.totaltypescript.com/tsconfig-cheat-sheet).

---

# Phase 1: Hono on Node (~3 days)

## Concepts that survived review

### 1. Web Fetch primitives + `@hono/node-server` bridge
- **Resource:** [Yusuke Wada — "Make Hono Work on Node.js" (Node Congress 2025, April 2025)](https://www.youtube.com/watch?v=4ks1RvEM99Y) + [Hono "Web Standards" docs](https://hono.dev/docs/concepts/web-standard).
- **Exercise (45 min, PASS):** Build a raw `http.createServer` that constructs a `Request` object manually from `IncomingMessage` and returns a `new Response(...)`. Replicate the same route using `@hono/node-server`. `curl` both and confirm headers and body match. This shows what the bridge actually is: a Fetch-to-Node adapter, mentally equivalent to Kestrel adapting raw sockets to ASP.NET's `HttpContext`.

### 2. Context (`c`) and typed `Variables`
- **Resource:** [Hono Context API docs](https://hono.dev/docs/api/context) + [Middleware guide](https://hono.dev/docs/guides/middleware) + [the typing-middleware-vars discussion](https://github.com/orgs/honojs/discussions/3257).
- **Exercise (60 min, PASS):** Build an `authMiddleware` using `createMiddleware<{ Variables: { userId: string } }>()` that reads `Authorization: Bearer`. Mount on a sub-router. Write Vitest tests via `app.request()` for header-present (200) and header-missing (401). Confirm TypeScript errors if a handler outside the middleware chain tries to read `c.var.userId`.

### 3. Onion middleware
- **Resource:** [Hono Middleware guide](https://hono.dev/docs/guides/middleware).
- **Exercise (45 min, REFRAMED to trigger the failure):** Build a `requestTimer` that stamps `performance.now()` before `next()` and computes elapsed after. Then deliberately call `next()` *twice* in a buggy middleware and watch the double-execution downstream. Fix it. The point is that middleware runs bidirectionally, not as a flat chain.

### 4. Routing + `AppType` export + RPC client
- **Resource:** [Hono Best Practices](https://hono.dev/docs/guides/best-practices) + [RPC guide](https://hono.dev/docs/guides/rpc) + [Yusuke's RPC blog post (Aug 2024)](https://blog.yusu.ke/hono-rpc/).
- **Exercise (60 min, REFRAMED):** Build a books CRUD router. Chain `.route('/books', books)` and assign to a declared variable. Export `AppType`. In a separate `client/` directory, instantiate `hc<AppType>('http://...')`. The non-obvious failure: modify the server's Zod schema, *don't* rebuild or re-export `AppType`, and watch the client silently send the wrong shape while the server rejects it at runtime. This is the seam to wire into your build and CI before it bites you in production.

### 5. Validation with `zod-validator` + Zod 4
- **Resource:** [Hono Validation guide](https://hono.dev/docs/guides/validation) + [`@hono/zod-validator` README](https://github.com/honojs/middleware/tree/main/packages/zod-validator). Confirmed: `@hono/zod-validator` 0.8.0 (May 2026) supports Zod 4.
- **Exercise (60 min, PASS):** Add `zValidator('json', CreateBookSchema)` to `POST /books`. Three Vitest cases: valid body (201), missing `title` (400), `year` as string (400). Confirm 400s come from the validator without custom error code.

### 6. ⭐ `HTTPException` + `onError` header-loss gotcha (TROPHY)
- **Resource:** [Hono HTTPException docs](https://hono.dev/docs/api/exception), the only place this gotcha is documented.
- **Exercise (60 min, KEEP, TROPHY):** Mount a CORS middleware that sets `Access-Control-Allow-Origin`. Throw `new HTTPException(403)` from a route. In `app.onError`, call `err.getResponse()` directly. Write a Vitest test asserting the CORS header survives, and watch it fail. Fix by cloning the response with context headers re-applied. The failure is non-obvious, and the fix teaches the immutable Response model that underlies all of Fetch.

### 7. DI by closures: the testing seam
- **Resource:** [Prisma + Hono guide](https://www.prisma.io/docs/orm/overview/databases/postgresql) + your .NET muscle memory.
- **Exercise (45 min, REFRAMED):** Build a `makeBooksRouter(db: PrismaClient)` factory. In production, instantiate one `PrismaClient` singleton and pass it in. In a Vitest, pass a hand-built mock object with the same shape (`{ book: { findMany: vi.fn() } }`). The takeaway isn't "use closures." It's that the testing seam in Hono is a function argument rather than a DI container, and the mock injection is what proves it.

### 8. Logging: pino + child logger middleware
- **Resource:** [pino docs](https://getpino.io) + Hono middleware docs. The `hono-pino` package is in security-only maintenance, so build from scratch.
- **Exercise (75 min, PASS):** Write a `loggerMiddleware` using `createMiddleware`: before `next()`, generate a `requestId` (`crypto.randomUUID()`), create a child logger (`rootLogger.child({ requestId })`), set both on context. After `next()`, log `{ method, url, status, durationMs }`. In `onError`, log at error level. Vitest: spy on pino transport, assert `requestId` in every log line for one request.

### 9. Graceful shutdown: make the race visible
- **Resource:** [Hono Node.js getting-started](https://hono.dev/docs/getting-started/nodejs).
- **Exercise (30 min, REFRAMED):** Add a slow endpoint with `await new Promise(r => setTimeout(r, 5000))`. `curl` it from one shell. From another, send `kill -SIGTERM <pid>`. Confirm the in-flight request completes (status 200, body returned) *before* the process exits. There's no `IHostedService` magic here: `server.close()` waits, and you call it manually.

### 10. Testing via `app.request()`
- **Resource:** [Hono Testing guide](https://hono.dev/docs/guides/testing) + [Testing Helper](https://hono.dev/docs/helpers/testing).
- **Exercise (60 min, PASS):** Full Vitest suite for books CRUD via `app.request()`. Then rewrite three tests using `hono/testing`'s `testClient`. Confirm zero ports are opened during `vitest run`. No supertest, no fixture server: that's the part worth getting used to.

## Killed from Phase 1
- **"Install fnm + pnpm" exercise (30 min):** nothing here a senior hasn't done a hundred times. Already in prereqs.
- **"Biome v2 noFloatingPromises" standalone (30 min):** fold it into the Vitest exercise. Write a floating promise, watch it pass silently in a test, then catch it with Biome.
- **"Vitest setup + coverage" (45 min):** 10 minutes for a senior. Subsumed.
- **"Zod env config" standalone (30 min):** a 5-minute pattern. Note it in your config file and move on.

---

# Phase 2: Prisma (~2 days)

The user's roadmap was right that Phase 2 is doing, not watching. Five Prisma docs pages cover 80% of the curriculum.

### 11. `schema.prisma` vs EF Core fluent API
- **Resource:** [Models](https://www.prisma.io/docs/orm/prisma-schema/data-model/models) + [Relations](https://www.prisma.io/docs/orm/prisma-schema/data-model/relations) (use unversioned `/docs/orm/` paths; the v6-pinned paths in older content document the previous major).
- **Exercise (45 min, REFRAMED):** Blog schema: `User`, `Post`, `Tag`, `PostTag` (explicit join with `@@id([postId, tagId])`), `Role` enum, `@@unique([email])` on User, `@@index([createdAt])` on Post. Run `prisma migrate dev --name init`. Open psql and confirm three things: the join table SQL has a composite PK, the enum became a `CHECK` constraint, and the index appears in `\d Post`. Reframed for .NET: notice you wrote zero C# config. The schema file *is* `OnModelCreating`.

### 12. ⭐ Migrate dev vs deploy + drift detection (TROPHY)
- **Resource:** [Mental model for Migrate](https://www.prisma.io/docs/orm/prisma-migrate/understanding-prisma-migrate/mental-model) + [Shadow database](https://www.prisma.io/docs/orm/prisma-migrate/understanding-prisma-migrate/shadow-database) + [Dev and production](https://www.prisma.io/docs/orm/prisma-migrate/workflows/development-and-production).
- **Exercise (60 min, TROPHY):** With the schema above applied, run `ALTER TABLE "Post" ADD COLUMN body_backup TEXT` directly in psql. Re-run `prisma migrate dev` and read the drift error verbatim. Drop the manual column. Then rename `Post.title` to `Post.headline` in `schema.prisma`, re-run, and read the generated `migration.sql` carefully: confirm it's `ALTER COLUMN ... RENAME`, not drop-and-add. The failure is self-inflicted, and the fix is just reading the SQL Prisma generated. The contrast with `dotnet ef database update` is hard to miss.

### 13. `findUnique` vs `findFirst`, `include`, N+1
- **Resource:** [Relation queries](https://www.prisma.io/docs/orm/prisma-client/queries/relation-queries) + [Query optimization](https://www.prisma.io/docs/orm/prisma-client/queries/query-optimization-performance).
- **Exercise (45 min, PASS):** Seed 10 users × 5 posts. Two versions: (a) `for (const u of users) { const posts = await u.posts(); }` (fluent), (b) `prisma.user.findMany({ include: { posts: { select: { title: true } } } })`. Enable `log: ['query']`. Count SQL lines: version A = 11 queries, version B = 1 or 2.

### 14. `relationLoadStrategy`: `join` vs `query`
- **Resource:** [Relation queries — relationLoadStrategy](https://www.prisma.io/docs/orm/prisma-client/queries/relation-queries). Still preview-flagged in May 2026 (`previewFeatures = ["relationJoins"]`).
- **Exercise (75 min, REFRAMED time):** Enable the preview flag. Query with `relationLoadStrategy: "join"`, capture the emitted SQL, and run it under `EXPLAIN ANALYZE` in psql to confirm `LATERAL` in the plan. Switch to `"query"` and confirm two separate SELECTs. **Prereq:** familiarity with reading `EXPLAIN ANALYZE`. Without it, this exercise is closer to 90 minutes.

### 15. ⭐ Interactive transactions: trigger the rollback (TROPHY)
- **Resource:** [Transactions](https://www.prisma.io/docs/orm/prisma-client/queries/transactions).
- **Exercise (45 min, TROPHY):** Implement transfer-funds: debit one account, credit another, throw if the balance would go negative. Write it first with `$transaction(async (tx) => {...})`. Throw deliberately mid-transfer, then query balances afterward and confirm both rows are unchanged. Now try the array form (`$transaction([q1, q2])`) and notice you cannot read the intermediate balance to guard against overdraft. The array form is for atomic sequencing only; the interactive form is for conditional logic. The critical footgun (use `tx.user.update` rather than `prisma.user.update` inside the callback, because escaping the `tx` handle means the write is uncommitted) becomes obvious once you trip it.

### 16. Next.js dev singleton
- **Resource:** [Next.js + Prisma troubleshooting](https://www.prisma.io/docs/orm/more/help-and-troubleshooting/help-articles/nextjs-prisma-client-dev-practices).
- **Exercise (20 min, PASS):** Remove the `globalForPrisma` singleton. Start `next dev`. Edit any file 10 times to trigger HMR. In psql, run `SELECT count(*) FROM pg_stat_activity WHERE application_name LIKE '%prisma%'` and watch the count climb with each save. Re-add the singleton, restart, and repeat: the count stays at 1.

### 17. Connection pooling: measure, don't trust
- **Resource:** [Database connections](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections).
- **Exercise (30 min, PASS):** In your Hono server, log on PrismaClient instantiation (use `log: ['query']` plus the constructor's lifecycle). Fire 100 concurrent requests with `autocannon -c 100 http://localhost:3001/notes`. Confirm "client instantiated" appears once while the query log fires 100 times. Then move `new PrismaClient()` inside the handler so instantiation fires 100 times, and watch Postgres reject new connections.

### 18. PgBouncer prepared-statement footgun
- **Resource:** [PgBouncer with Prisma](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections/pgbouncer). Note that PgBouncer ≥1.21 now supports prepared statements natively, so the `?pgbouncer=true` flag is soft-deprecated for new setups.
- **Exercise (90 min, LIE→corrected time):** Spin up PgBouncer < 1.21 (or 1.21+ with `max_prepared_statements = 0`) via a docker-compose file (provided in your prereqs). Connect Prisma without `?pgbouncer=true`. Run a Prisma interactive transaction with two statements and observe `prepared statement does not exist`. Add `?pgbouncer=true` and repeat: the error disappears because prepared statements are now disabled. The honest time reflects reality, since the Docker compose plus PgBouncer config alone is 30+ minutes.

### 19. Raw SQL: `$queryRaw` and TypedSQL
- **Resource:** [Raw queries](https://www.prisma.io/docs/orm/prisma-client/using-raw-sql/raw-queries) + [TypedSQL](https://www.prisma.io/docs/orm/prisma-client/using-raw-sql/typedsql) (preview).
- **Exercise (45 min, PASS):** Write a query Prisma's fluent API can't express, e.g. `ROW_NUMBER() OVER (PARTITION BY authorId ORDER BY createdAt DESC)`. Do it once with `$queryRaw` plus a manual type annotation, then once with TypedSQL (`.sql` file + `prisma generate --sql`). Confirm that TypedSQL gives full autocomplete, `$queryRaw` requires manual type maintenance, and the emitted SQL is identical under `EXPLAIN`.

### 20. EF Core mental shift: observe the inefficiency
- **Resource:** No 2024-2026 EF→Prisma guide is great. Build it yourself; the [DEV Community post (2022)](https://dev.to/prisma/database-migrations-for-net-and-entity-framework-with-prisma-49e0) is conceptual scaffolding.
- **Exercise (60 min, REFRAMED to trigger the failure):** Translate an "update user profile" handler. First write the EF-style pattern in Prisma: `findUnique`, mutate object fields locally, then `update({ where, data: object })`. Measure via the query log and you'll see two queries (read plus write). Now collapse it to `prisma.user.update({ where: { id }, data: { name, updatedAt: new Date() } })` directly, which is one query. Seeing the extra round trip is what makes the difference stick.

### Killed/replaced from Phase 2
- The `practica.dev` "Is Prisma better than your traditional ORM?" link cited by the research agent renders blank. Skip it.

---

# Phase 3: Next.js 16 App Router + RSC (~4 days, the biggest phase)

### 21. ⭐ RSC vs Client + serializability: break it with Prisma `Decimal` (TROPHY)
- **Resource:** [Server and Client Components docs (v16.2.6)](https://nextjs.org/docs/app/getting-started/server-and-client-components) + [`use cache` API reference](https://nextjs.org/docs/app/api-reference/directives/use-cache) for the canonical type-support list.
- **Exercise (45 min, TROPHY):** Build a `<ProductCard>` Server Component that fetches a product via Prisma (price field is `Decimal`). Pass the raw Prisma object directly to an `<AddToCart>` `'use client'` component as a prop and observe the runtime/build error. Fix it by converting `Decimal` to `number` (or using superjson) at the boundary. Add a `Date` field and confirm it survives. This is exactly the bug you'll hit in week two of real work, and getting it wrong once makes the serialization boundary stick.

### 22. Server children through Client Component pattern
- **Resource:** Same docs page, "Interleaving Server and Client Components" section + [composable caching blog](https://nextjs.org/blog/composable-caching).
- **Exercise (30 min, REFRAMED verification):** Build a `<SidebarShell>` Client Component with `useState` for collapse/expand. Pass a `<UserActivity>` RSC (which `await`s a Prisma call) as `children`. To verify, open the Network tab in DevTools, find the `?_rsc=` request, and confirm `<UserActivity>`'s output is in the RSC payload rather than the client bundle. (This is more accessible than React DevTools' "rendered by" tree.)

### 23. Data-read decision tree (RSC await vs TanStack Query vs `'use cache'`)
- **Resource:** [Caching docs](https://nextjs.org/docs/app/getting-started/caching) + [Vercel Academy — Cache Components lesson](https://vercel.com/academy/nextjs-foundations/cache-components).
- **Exercise (90 min, LIE→corrected):** Build `/products` with three data shapes: (a) `<Header>` RSC `await`ing one-off config (no cache), (b) `<ProductGrid>` with `'use cache'` + `cacheLife('hours')` + `cacheTag('products')`, (c) `<StockBadge>` Client Component using TanStack Query polling every 30s. Run `next build`, inspect output to confirm ProductGrid is in the static shell and StockBadge is not. **Prereq:** the Phase 1 Hono server running on :3001 with seeded products.

### 24. Data-write decision tree: force a stale-data failure
- **Resource:** [Building APIs with Next.js (Feb 2025)](https://nextjs.org/blog/building-apis-with-nextjs) + [Data Security](https://nextjs.org/docs/app/guides/data-security).
- **Exercise (90 min, LIE→corrected):** Build a publish-post flow: `<PostForm>` Client to a Server Action calling Hono to `updateTag('posts')`. Then build a Route Handler at `/api/webhooks/cms` that calls `revalidateTag('posts', 'max')` (use the two-arg form, since single-arg is deprecated). To trigger the failure, deliberately omit `updateTag()` in the Server Action, submit the form, and observe the stale list. Add `updateTag()` back and you get read-your-writes. Mutation without cache invalidation leaves stale UI no matter how clever the action is.

### 25. ⭐ Route protection layers + curl bypass (TROPHY)
- **Resource:** [Next.js 16 blog](https://nextjs.org/blog/next-16) + [Upgrading to v16](https://nextjs.org/docs/app/guides/upgrading/version-16) + [Data Security docs](https://nextjs.org/docs/app/guides/data-security).
- **Exercise (45 min, TROPHY):** Three layers protecting `/dashboard`: (1) `proxy.ts` redirects unauthenticated requests to `/login`, (2) `layout.tsx` calls `await auth()` and `redirect()`, (3) the `deleteRecordAction` Server Action calls `await auth()` and throws if null. Now craft a `curl -X POST` directly against the Server Action's POST endpoint (you'll need to extract the action ID, see exercise #28). The curl request bypasses the proxy entirely because it doesn't follow the route matcher, and the in-action `auth()` check is the only thing that saves you. Middleware controls routing, not security.

### 26. Next.js 16 landmines
- **Resource:** [Upgrading to v16](https://nextjs.org/docs/app/guides/upgrading/version-16), the full breaking-changes list.
- **Exercise (30 min, REFRAMED):** Scaffold a minimal Next 15 starter (provided as a Gist or in your prereqs; don't build it cold). Run `npx @next/codemod@canary upgrade latest`. Manually verify async `params`. Add `cacheComponents: true`. The failure to watch for: a component that calls `cookies()` without being inside `<Suspense>` now errors with `Uncached data accessed outside <Suspense>`. Wrap it, then confirm `next build` succeeds.

### 27. Pre-existing Next 15 shifts: make staleness visible
- **Resource:** [Our Journey with Caching (Oct 2024)](https://nextjs.org/blog/our-journey-with-caching) + [Caching without Cache Components](https://nextjs.org/docs/app/guides/caching-without-cache-components).
- **Exercise (30 min, REFRAMED, failure-first):** In an RSC, write `const data = await fetch('http://localhost:3001/products')`. Seed a product priced at $10 and render the price. Update the price to $20 directly in Postgres, then reload the page. The price reflects $20, because fetch is *not* cached by default in Next 15+. Now add `{ cache: 'force-cache' }` and reload: stale. Add `revalidateTag` and tag the fetch, reload again: fresh. Opt-in caching means surprise dynamics until you ask for caching.

### 28. ⭐ Server Actions security: grep your own build (TROPHY)
- **Resource:** [Data Security docs](https://nextjs.org/docs/app/guides/data-security) + [Sebastian Markbåge's security post (Oct 2023, still authoritative)](https://nextjs.org/blog/security-nextjs-server-components-actions) + [Arcjet's plain-language take (Nov 2024)](https://blog.arcjet.com/next-js-server-action-security/).
- **Exercise (45 min, TROPHY):** Export a Server Action. Run `next build`. Grep `.next/static` for `__ACTION_` to find the action ID hash. Construct a `curl -X POST` against your app's POST endpoint with the action ID in the header and fabricated `FormData`. It succeeds if the action doesn't call `auth()`. Now add `auth()` plus an authorization check inside the action and re-curl: 401. Add `serverActions.allowedOrigins` to `next.config.ts`, then curl from a "wrong" Origin header: 403. This is what teaches that actions are public POST endpoints.

### 29. Senior-React traps under RSC (split into two)
Reviewer A killed the original 5-trap "gauntlet" as pedagogically incoherent. Two focused exercises instead:

**29a. Server-only context + async cookies (30 min, REFRAMED):**
Wrap the root layout in a `<ThemeContext.Provider value={theme}>` where `theme = await cookies().then(c => c.get('theme'))`. Observe the error: `cookies()` is a runtime API and must be in a `<Suspense>` or `'use cache': private` scope. Fix by hoisting the cookies read out of the cached scope and passing `theme` as a prop to a small Client Component that supplies the Provider deeper in the tree. **Bonus:** push the Provider out of the root layout entirely if `useTheme()` is only needed below `/app/dashboard/*`, and watch the static shell of `/` recover its prerender.

**29b. Env var leakage (30 min, REFRAMED to diagnosis):**
In a `'use client'` component, write `console.log(process.env.DB_SECRET)`. Build, open the browser, and you'll see `""`, because Next silently replaces non-`NEXT_PUBLIC_` vars. Add `next/bundle-analyzer` and confirm the secret isn't in the bundle (Next did the right thing). Then write the same code in an RSC and observe the real value. Add `import 'server-only'` to your DB module, try to import it from a Client Component, and read the build-time error. The senior skill here is the diagnosis: using the bundle analyzer to *prove* a secret isn't shipped.

### 30. Cache Components debug: `Uncached data accessed outside <Suspense>`
- **Resource:** [Caching getting-started](https://nextjs.org/docs/app/getting-started/caching) + [`use cache` troubleshooting](https://nextjs.org/docs/app/api-reference/directives/use-cache#troubleshooting). Set `NEXT_PRIVATE_DEBUG_CACHE=1` for verbose cache logging.
- **Exercise (45 min, PASS):** With `cacheComponents: true`, add `await cookies()` directly in a component with no Suspense wrapper and read the dev error verbatim. Fix it with `<Suspense fallback={...}>`, then compare against fixing it with `'use cache': private`. Run `next build` and confirm the static shell output.

### Killed from Phase 3
- **Standalone `server-only` import test (20 min):** folded into 29b above.
- **The Jack Herrington video:** the research agent couldn't verify the upload date, and it's only useful if post-Oct 2025. Check the video's date yourself before relying on it.
- **`nextjs.org/learn/dashboard-app`:** useful as scaffolding, but it predates Next 16 conventions. Don't trust its caching or proxy patterns, and overlay Vercel Academy's Cache Components lesson instead.

---

# Phase 4: Auth.js v5 (~2 days)

**2026 context (do not skip):** Auth.js was handed to the Better Auth team on Sep 22, 2025. authjs.dev shows a banner, and the official site hosts a [migrate-to-Better-Auth guide](https://authjs.dev/getting-started/migrate-to-better-auth). For greenfield work, prefer Better Auth. Your team is on Auth.js, so you learn it as-shipped.

### 31. `auth.ts` + GitHub provider scaffold (honest time)
- **Resource:** [authjs.dev installation](https://authjs.dev/getting-started/installation).
- **Exercise (60 min, LIE→corrected):** Scaffold Next.js, register a GitHub OAuth App if you didn't in prereqs (~15 min cold), and wire GitHub-only Auth.js. Export `{ handlers, auth, signIn, signOut }` from `auth.ts`. Mount at `app/api/auth/[...nextauth]/route.ts`. Confirm `/api/auth/providers` returns GitHub and `/api/auth/signin` shows the button. Reframed from 30 minutes, because GitHub OAuth registration alone eats half the original budget.

### 32. `auth()` everywhere + the missing-`auth()` failure
- **Resource:** [Session management — getSession](https://authjs.dev/getting-started/session-management/get-session).
- **Exercise (30 min, REFRAMED to include a failure):** Build three pages (an RSC, a Server Action, and a Route Handler) that each call `await auth()`. Then strip the `auth()` call from the Server Action, trigger it from an unauthenticated client (curl, per #28's playbook), and watch the action still fire. Re-add the check and you get 401. `auth()` isn't a wrapper. It's a check you must invoke in every entry point.

### 33. Edge config split
- **Resource:** [Edge compatibility](https://authjs.dev/guides/edge-compatibility).
- **Exercise (45 min, PASS):** Add the Prisma adapter to `auth.ts`. Create `auth.config.ts` with no DB imports. Have `middleware.ts` / `proxy.ts` import only from `auth.config.ts`. Break it by importing `auth.ts` in middleware and read the edge-incompatibility error verbatim, then fix it. Note that if your Next is 16+, `proxy.ts` runs on the Node runtime, so the split may be theoretically unnecessary. It's still good hygiene, and it's required if any code path falls back to Edge.

### 34. Session strategy: JWT vs database
- **Resource:** [Concepts overview](https://authjs.dev/concepts) + [Prisma adapter](https://authjs.dev/getting-started/adapters/prisma).
- **Exercise (30 min, REFRAMED):** Branch the project. Set `session: { strategy: 'jwt' }` in one branch and `'database'` in the other. Instrument the `jwt` callback with a counter that logs every fire. Drive the app: sign in, refresh, navigate, call an RSC. On the JWT branch the counter fires on every request; on the database branch it fires only on sign-in. The strategy decision comes down to "do I want a DB hit per session read?", and this lets you watch the answer rather than read about it.

### 35. Prisma adapter schema + Account vs Session table
- **Resource:** [Prisma adapter docs](https://authjs.dev/getting-started/adapters/prisma).
- **Exercise (30 min, PASS):** Migrate the adapter schema. Sign in with GitHub. In psql, inspect the `Account` table and confirm `access_token` is stored. Toggle the session strategy to JWT and re-sign-in: the `Session` table is empty while `Account` is still populated.

### 36. ⭐ The callbacks pattern: flip the order
- **Resource:** [Extending the session](https://authjs.dev/guides/extending-the-session) + [Role-based access control](https://authjs.dev/guides/role-based-access-control).
- **Exercise (45 min, KEEP, near-trophy):** Add `role` to `User`. In the `jwt` callback: `if (user) token.role = user.role`. In the `session` callback: `session.user.role = token.role`. Verify `await auth()` returns `session.user.role`. Now flip the order, setting `role` only in `session` and not in `jwt`, and you'll get `undefined`. The session callback can only see what the jwt callback put on `token`.

### 37. ⭐ JWE vs JWS: getting the two decoders to disagree
- **Resource:** [WorkOS JWE/JWS/JWK/JWKS primer (March 2025)](https://workos.com/blog/jws-jwe-jwk-jwks-explained) for theory, and [authjs.dev/reference/core/jwt](https://authjs.dev/reference/core/jwt) for the `decode({ token, secret, salt })` signature.
- **Exercise (10 min, REFRAMED to cut dot-counting):** Grab the raw `authjs.session-token` cookie. Try `jwt.verify(cookie, AUTH_SECRET)` from `jsonwebtoken` and watch it fail, since it can't parse a 5-part JWE. Now call `decode({ token, secret, salt: 'authjs.session-token' })` from `@auth/core/jwt` and it succeeds. This is where Phase 5 starts to make sense. Note the `salt` parameter: the research agents originally omitted it, and you'd hit a confusing decode error without it.

### Killed from Phase 4
- **`AUTH_SECRET` rename exercise (15 min):** pure ceremony with nothing to learn. Note it in your env config and move on.
- **JWE dot-counting:** folded into #37, where the actual decoder failure carries the lesson.

---

# Phase 5: Cross-service auth, Next ↔ Hono (~3 days, time-corrected from 2)

This is the hardest phase. Reviewer A re-ordered the options so the "forward session cookie" approach is presented last, as an escape hatch rather than a starting point.

### Pre-reading (mandatory, 30 min)
[WorkOS — JWE, JWS, JWK, JWKS explained](https://workos.com/blog/jws-jwe-jwk-jwks-explained), the single best primer for a senior. Read it before any exercise here.

### 38. Option 2a: HS256 + shared secret (the baseline that works)
- **Resource:** Next.js + `jose` for signing; [Hono JWT middleware](https://hono.dev/docs/middleware/builtin/jwt) for verification.
- **Exercise (45 min, PASS):** In the Auth.js `session` callback, mint a short-lived HS256 JWT (60s `exp`) signed with `process.env.API_SECRET`, *not* `AUTH_SECRET`, since they're different trust boundaries. Stamp `sub` and `roles`, and store the token on `session.apiToken`. The RSC reads `session.apiToken` and calls Hono with `Authorization: Bearer`, and Hono verifies with `jwt()` middleware using the same `API_SECRET`. Then trigger the failures in turn: sign with the wrong secret on the Hono side and watch it 401, let the token expire and watch it 401, pass a tampered token and watch it 401. The caveat is that a shared secret makes rotation painful. It's acceptable for single-app internal use, but don't reach for it beyond that without thinking.

### 39. ⭐ Option 2b: RS256 + JWKS + rotation (honest time = 120 min)
- **Resource:** [WorkOS — Verifying JWTs in Next.js App Router](https://workos.com/blog/how-to-verify-jwts-in-nextjs-app-router) + [Hono JWK middleware docs](https://hono.dev/docs/middleware/builtin/jwk). **Security:** ensure Hono ≥ 4.11.4 (advisory [GHSA-3vhc-576x-3qv4](https://github.com/honojs/hono/security/advisories/GHSA-3vhc-576x-3qv4), CVE-2026-22818). Always pin `alg: ['RS256']` server-side.
- **Exercise (120 min, LIE→corrected, near-trophy):**
  - **Stage 1 (60 min):** Generate an RSA key pair with `openssl genrsa`. Don't generate at runtime, because Next.js restarts lose the key; store the private and public keys as PEM strings in env vars. Implement an `/api/jwks` Route Handler returning `{ keys: [exportedJwk] }` via `jose.exportJWK`. In the `session` callback, sign with `new SignJWT({ sub, roles })`, set the protected header `{ alg: 'RS256', kid: 'k1' }`, and set `iss`, `aud: 'hono-api'`, and a 5-min `exp`. The RSC calls Hono with the resulting Bearer, and Hono's `jwk()` middleware with `jwks_uri` + `alg: ['RS256']` verifies it. Confirm it works end-to-end.
  - **Stage 2 (60 min), rotation:** Generate a second key pair (`kid: 'k2'`). Expose both in JWKS and start signing with `k2`. Confirm old `k1`-signed tokens still verify, since they're still in JWKS. Remove `k1` from JWKS once the longest-lived `k1` token has expired. This is where the `kid` field earns its keep.

### 40. Option 1: forward session cookie + `@auth/core/jwt` decode (the escape hatch, presented last)
- **Resource:** [authjs.dev/reference/core/jwt](https://authjs.dev/reference/core/jwt). The docs warn this module "will be refactored/changed", so don't build production on it.
- **Exercise (30 min, PASS, presented last):** In a Server Action, read the `authjs.session-token` cookie and POST it to Hono in a custom header. On Hono, call `decode({ token, secret: process.env.AUTH_SECRET, salt: 'authjs.session-token' })`. This sits last for a reason, per Reviewer A: learn it first and you stop at "it works" and never reach the proper JWKS pattern. Treat it as a last-resort prototype tool only.

### 41. ⭐ Server-only model breaks: SSE from Hono
- **Resource:** No single resource covers this for the Next+Hono pairing in 2026. Build it.
- **Exercise (45 min, PASS, near-trophy):** Build a Hono SSE endpoint (`text/event-stream`, pushing messages on a 1s interval). Try to consume it from a Server Action and it fails, because SSE is browser-direct and you can't stream back through an action. Switch to a `<LiveFeed>` Client Component using `EventSource`, and now the browser has no Bearer token to pass. You need to implement the resolution pattern here, not just describe it: add an `/api/sse-token` route in Next that calls `auth()` server-side and returns a short-lived token (in-memory cache or signed JWT). The Client Component fetches that first, then opens `EventSource` with `Authorization: Bearer`. Because `EventSource` doesn't natively support custom headers in the browser, fall back to either the `event-source-polyfill` package or a `?token=` query param Hono can validate. The whole "server-to-server only" story collapses the first time a real-time feature shows up.

### 42. CORS / CSRF realities
- **Resource:** [Hono CORS middleware](https://hono.dev/docs/middleware/builtin/cors).
- **Exercise (20 min, PASS):** Call your Hono endpoint from a Client Component with `fetch('http://localhost:3001/products', { credentials: 'include' })` and observe the CORS preflight failure. Add Hono `cors({ origin: 'http://localhost:3000', credentials: true, allowHeaders: ['Authorization', 'Content-Type'] })` and confirm it works. Then try `origin: '*'` with `credentials: true` and the browser blocks it, because the spec forbids that combination.

### Mostly-skip
- **Option 3, full OAuth resource server (Keycloak/Auth0/WorkOS):** out of scope for a single-team ramp. It's the right answer for multi-app or multi-team setups, but don't build it during the 16-day plan.
- **Option 4, Better Auth:** know it exists and watch [devtools.fm Ep. 150 — Bereket Engida](https://www.devtools.fm/episode/150). Don't migrate during the ramp.

---

# Phase 6: Production hardening (concepts already in ROADMAP)

ROADMAP.md covers this well, so re-read it. Key resources are unchanged:

- **Self-hosting:** [Lee Robinson — "Self-Hosting Next.js" (Oct 2024)](https://www.youtube.com/watch?v=sIVL4JMqRfc) plus the repo. The deployment mechanics are unchanged.
- **Cache Components debug + Suspense holes:** the Vercel Academy lesson already used above.
- **OTEL:** the `pino-opentelemetry-transport` README. Read it when you wire it.

There are no new exercises here. Phase 1's logging exercise (#8) is the seed; production-harden it by adding OTEL propagation across the Next-to-Hono boundary when you actually deploy.

---

# What didn't survive review (transparency)

Cutting weak material is part of the job. Here are the specific kills, each with the strongest argument for keeping it that nonetheless lost:

| Killed | Counter-argument | Why it still lost |
|---|---|---|
| "Install fnm + pnpm" exercise | New ecosystem might surprise | A senior installs runtimes in their sleep. Nothing to learn. |
| Standalone Vitest setup | Vitest API differs slightly from Jest | 10 minutes for a senior. Fold into the first real exercise. |
| Standalone Zod env config | Beginners forget Zod parse-at-boot | Mention it in your config file. Not 30 minutes of work. |
| `AUTH_SECRET` rename | Some teams still on `NEXTAUTH_*` | Three minutes. Not an exercise. |
| `jwt.verify` dot-counting | Concrete artefact of JWE | The decoder failure (#37) teaches the same thing with a real error. |
| 5-traps-in-one RSC gauntlet | "Throughput" | Pedagogically incoherent: learners forget trap 1 by trap 3. Split into two. |
| `practica.dev` Prisma article | Sharp criticism | The page renders blank in May 2026. |
| Forward-session-cookie as Option 1 | "It works" | Teaching it first short-circuits the learning. Moved to last. |
| Phase 2 video courses | Variety | The Prisma docs are best-in-class. Save the dollars. |

---

# The single most-damaging assumption

The original research assumed that "understand the concept" equals "run the happy path."

If you build a Hono middleware and it works, you've proven the syntax, not the model. If you wire Auth.js and the session reads, you've proven the wiring, not the JWE semantics. The exercises that teach a senior dev are the ones that force the failure first. Drift detection breaks `migrate dev`. Prisma `Decimal` breaks RSC serialization. `jwt.verify` breaks on a JWE. A `curl` bypasses your `proxy.ts`. Three exercises (#12, #21, #25/#28) set the bar, and the rest of this file was reshaped to match that template.

If you're tempted to skip an exercise because "I get it conceptually," that's exactly the signal to do it. Concept understanding without the failure observed never sticks.

---

# The reading list, final and ordered

Below is the 80% list. Everything else is supplemental.

**Phase 1 / Hono (read once, then build):**
1. [Hono Best Practices](https://hono.dev/docs/guides/best-practices) + [Validation](https://hono.dev/docs/guides/validation) + [Testing](https://hono.dev/docs/guides/testing) + [RPC](https://hono.dev/docs/guides/rpc) + [Context](https://hono.dev/docs/api/context) + [HTTPException](https://hono.dev/docs/api/exception).
2. [Yusuke Wada — "Make Hono Work on Node.js"](https://www.youtube.com/watch?v=4ks1RvEM99Y) (15 min).
3. [Matt Pocock — TSConfig cheat sheet](https://www.totaltypescript.com/tsconfig-cheat-sheet).

**Phase 2 / Prisma:**
4. [Prisma Migrate mental model + Shadow DB](https://www.prisma.io/docs/orm/prisma-migrate/understanding-prisma-migrate/mental-model).
5. [Relation queries](https://www.prisma.io/docs/orm/prisma-client/queries/relation-queries) + [Transactions](https://www.prisma.io/docs/orm/prisma-client/queries/transactions).
6. [PgBouncer footguns](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections/pgbouncer).

**Phase 3 / Next.js:**
7. [Server and Client Components (v16.2.6)](https://nextjs.org/docs/app/getting-started/server-and-client-components).
8. [Caching getting-started](https://nextjs.org/docs/app/getting-started/caching) + [`use cache` API reference](https://nextjs.org/docs/app/api-reference/directives/use-cache).
9. [Data Security](https://nextjs.org/docs/app/guides/data-security).
10. [Upgrading to v16](https://nextjs.org/docs/app/guides/upgrading/version-16), the full breaking-changes reference.
11. [Vercel Academy — Cache Components](https://vercel.com/academy/nextjs-foundations/cache-components) (the single confirmed-current paid-style course).

**Phase 4 / Auth.js:**
12. [authjs.dev installation + adapters/prisma + extending-the-session + role-based-access-control](https://authjs.dev/getting-started/installation).
13. [Better Auth handover blog](https://better-auth.com/blog/authjs-joins-better-auth), context for why this stack is in maintenance.

**Phase 5 / Cross-service:**
14. [WorkOS — JWE/JWS/JWK/JWKS](https://workos.com/blog/jws-jwe-jwk-jwks-explained) (theory, 20 min).
15. [WorkOS — Verifying JWTs in Next.js App Router](https://workos.com/blog/how-to-verify-jwts-in-nextjs-app-router) (implementation).
16. [Hono JWK middleware docs](https://hono.dev/docs/middleware/builtin/jwk); ensure ≥ 4.11.4 (CVE-2026-22818).

One paid month covers the only paid-worthy slot: Frontend Masters' [Next.js Fundamentals v4 (Scott Moss)](https://frontendmasters.com/courses/next-js-v4/). Overlay Vercel Academy's Cache Components lesson on top, since the Frontend Masters course is pre-Next-16.

---

# Total honest time budget

| Phase | Exercises kept | Honest hours |
|---|---|---|
| 0 (toolchain) | 1 (tsconfig) | ~2 |
| 1 (Hono) | 10 | ~9 |
| 2 (Prisma) | 9 (with PgBouncer at 90 min) | ~7 |
| 3 (Next.js) | 9 (corrected times) | ~10 |
| 4 (Auth.js) | 7 | ~5 |
| 5 (Cross-service) | 5 (JWKS at 120 min, SSE-break at 45) | ~6 |
| **Total** | **41 exercises** | **~39 hours** |

At 2 hours/day, ~20 working days. ROADMAP.md's 16-day estimate was tight; this corrects to ~4 weeks. At 4 hours/day, 10 working days. Plan accordingly.
