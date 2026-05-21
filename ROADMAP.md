# Next.js + Hono + Prisma Migration Roadmap
**For a senior C# / .NET + functional React/TS developer**
*Generated 2026-05-21 via a four-researcher / three-reviewer agent team*

---

## Read this first: the 2026 landscape (what's stale, what's new)

Your training data and most tutorials will quietly mislead you on these:

- **Next.js 16 (current)** — `middleware.ts` was renamed to `proxy.ts` (codemod handles it). `proxy.ts` defaults to the Node.js runtime; Edge support stays in `middleware.ts`. `params`, `searchParams`, `cookies()`, `headers()` are now async (`await` them). PPR-as-a-concept is **gone** — folded into Cache Components (`cacheComponents: true` + `'use cache'` + `cacheLife()`/`cacheTag()`). `revalidateTag(tag, profile)` requires a cache-life profile; single-arg form deprecated; `updateTag()` exists for read-your-writes in Server Actions.
- **Next.js 15 already shifted defaults** — `fetch` is no longer auto-cached; GET Route Handlers are dynamic by default.
- **Auth.js / NextAuth** — **Auth.js was handed over to the Better Auth team in September 2025. v5 never reached GA. The official Auth.js site now hosts a "migrate to Better Auth" guide.** Your project uses it, so you must learn it. But if your team starts a fresh service, push back on Auth.js as the choice. Sources: [better-auth.com/blog/authjs-joins-better-auth](https://better-auth.com/blog/authjs-joins-better-auth) ; [authjs.dev/getting-started/migrate-to-better-auth](https://authjs.dev/getting-started/migrate-to-better-auth).
- **Node 24 LTS** + native TS type stripping is **stable** in 24.3+/22.18+. `require(esm)` is stable since late 2025 (22.12+/23+). The "ESM-only purity" argument is weaker than older posts claim.
- **Biome v2** shipped `noFloatingPromises` in nursery (~75% parity with typescript-eslint). The common "Biome can't do type-aware rules" line is now outdated.

---

## What to SKIP (your existing skills already cover this)

Don't waste course time on:

- React fundamentals — hooks, controlled forms, Context, Error Boundaries, Suspense semantics.
- TS language features — structural typing, discriminated unions, `satisfies`, `as const`, generics, `unknown` vs `any`.
- async/await mechanics — map `CancellationToken` muscle memory to `AbortSignal`.
- HTTP basics, JWT signing/verification fundamentals, OIDC concepts.
- DI as a concept — you'll be *unlearning* (closures replace containers), not learning.
- Node internals theater — event-loop phases, `nextTick` vs `setImmediate`, `worker_threads`, Buffer vs Uint8Array. Read one paragraph on each: "don't block the loop, offload to a queue". That's enough.
- pnpm as a topic — it's `npm` with a strict lockfile and a `deploy` command. 10 minutes.

---

## The roadmap — 8 phases, ~6–8 weeks at a few hours/day

### Phase 0 — Toolchain (1 day)

Install Node 24 LTS (via `nvm` or `fnm`), enable pnpm 10 via `corepack enable`, scaffold a Hono service with `pnpm create hono@latest`.

**Strict production tsconfig.json** (set these flags from day one — adding them later is a migration project):

```jsonc
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "outDir": "dist"
  }
}
```

Pick **one** lint/format stack and move on. Biome v2 is fine for new code; ESLint flat config + `typescript-eslint` is fine if you want maximum type-aware rule coverage. Don't teach yourself both.

**Exercise:** scaffold a Hono service, write `GET /health` returning `{ ok: true }`, write a Vitest that asserts via `app.request('/health')`.

---

### Phase 1 — Hono on Node, end-to-end (3 days)

This is your daily-driver framework. Learn it before Next.js.

**Concepts that matter:**

- **Web Fetch primitives everywhere** — Hono builds on `Request`/`Response`/`Headers`/`URL`. `@hono/node-server` bridges to Node's `http`.
- **Context (`c`)** — `c.req.valid('json')`, `c.json()`, and `c.set/c.get/c.var` for request-scoped state (lifetime = one request). Type per-app with `new Hono<{ Variables: {...} }>()` — avoid the global `ContextVariableMap` augmentation, it masks runtime `undefined`.
- **Onion middleware** — code before `await next()` runs inbound, code after runs outbound. Bidirectional wrap. Not Express's flat chain.
- **Routing layout** — inline handlers, then split features via `app.route('/books', books)`. Don't write controller classes — type inference dies. Chain returns and `export type AppType = typeof app` so the `hc<AppType>('http://...')` RPC client gives you Refit-style end-to-end types.
- **Validation** — `@hono/zod-validator` with Zod 4 schemas. `c.req.valid('json')` returns the typed, parsed value.
- **Error handling** — `throw new HTTPException(401, { message: 'Unauthorized' })`, register `app.onError(...)`. **Gotcha:** `HTTPException.getResponse()` does NOT include headers you set on `c` — re-apply them in `onError`. There is no global filter pipeline; `app.onError` *is* it.
- **DI by closures** — module-level singletons for app-scoped (`export const prisma = ...`); `c.set('user', ...)` in middleware for request-scoped. There is no `IServiceCollection`. Don't reach for awilix/tsyringe until you actually need cross-module testability you can't get with closures.
- **Config** — read `process.env`, validate with Zod at boot, export a typed object. That's your `IOptions<T>` — and it crashes on boot if a var is missing.
- **Logging** — `pino` + a child logger middleware that stamps `requestId`/`userId` on every line.
- **Graceful shutdown** — `serve({ fetch: app.fetch, port })`, listen for `SIGINT`/`SIGTERM`, call `server.close()`. Manual — no `IHostedService`.
- **Testing** — `await app.request('/path', init)`. No port, no supertest, no fixture server. Killer feature.

**ASP.NET Core ↔ Hono mapping:**

| ASP.NET Core | Hono / Node |
|---|---|
| `IServiceCollection` | closures + module exports; `c.set/c.var` for request scope |
| `app.Use(async (ctx, next) => …)` | `app.use(async (c, next) => …)` — onion |
| `[FromBody]` + FluentValidation | `zValidator('json', schema)` + `c.req.valid('json')` |
| `IConfiguration` + `IOptions<T>` | Zod-validated `process.env` at boot |
| `ILogger<T>` | pino + child logger via middleware |
| `IHostedService` / `IHostApplicationLifetime` | `serve()` + manual SIGINT/SIGTERM |
| `IExceptionFilter` / `UseExceptionHandler` | `app.onError(...)` + `HTTPException` |
| `WebApplicationFactory` + `HttpClient` | `app.request()` directly |
| `CancellationToken` | `AbortSignal` (`c.req.raw.signal`) |
| Refit / NSwag-generated clients | `hc<AppType>('http://...')` RPC client |

**Exercise:** build `/notes` CRUD with Zod-validated bodies, pino logging, `app.onError` returning a typed error envelope, and Vitest tests via `app.request()`. Export `AppType` and consume it from a tiny script using `hc`.

---

### Phase 2 — Prisma (2 days)

**Concepts that matter:**

- **`schema.prisma`** — models, `@relation`, `@@id`, `@@unique`, `@@index`, enums. One file, declarative. No Fluent API.
- **Migrations** — `prisma migrate dev` (autogenerates + applies + uses a *shadow database* to detect drift between recorded migrations and actual schema). `prisma migrate deploy` is production-only: applies pending migrations, never resets, holds an advisory lock. **Drift** = schema changed outside migration history; `dev` will prompt to reset, `deploy` will error. Run `deploy` from CI only.
- **Client** — `findUnique` (PK / unique field only, batchable) vs `findFirst` (any filter). `select` whitelist projection; `include` pulls relations. Can nest `select` inside `include`. N+1: prefer `include`/`select` over chained `.posts()` (the chained form issues two queries).
- **`relationLoadStrategy`** — `"join"` uses LATERAL JOIN + JSON aggregation in one round trip; `"query"` issues separate queries + stitches in JS. **Caveat:** still behind the `relationJoins` preview flag — don't assume it's the default in your codebase; check `previewFeatures` in `schema.prisma`.
- **Transactions** — `prisma.$transaction([q1, q2])` (sequential, all-or-nothing, no conditional logic) or `prisma.$transaction(async (tx) => { ... })` (interactive — pass `tx` to every op inside). Options: `{ isolationLevel, maxWait, timeout }`. Keep them short; they hold a connection.
- **Global singleton in Next.js dev** (mandatory — without this, hot reload exhausts your DB pool within minutes):

  ```ts
  const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };
  export const prisma = globalForPrisma.prisma ?? new PrismaClient();
  if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma;
  ```

- **Connection pooling** — pool opens lazily on first query. In long-running Node (your Hono), reuse the singleton. In serverless, every cold start is a fresh pool; front with PgBouncer or Accelerate.
- **PgBouncer transaction mode** — append `?pgbouncer=true`, split `DATABASE_URL` (pooled, for the client) and `DIRECT_URL` (direct, for migrate). **Footguns:** transaction mode breaks prepared statements, `LISTEN/NOTIFY`, advisory locks, `SET LOCAL`, and Prisma interactive transactions. Plan around it.
- **Raw SQL** — `prisma.$queryRaw\`SELECT * FROM "User" WHERE id = ${id}\`` is parameterized and safe. `Prisma.sql` + `Prisma.join` for composition. TypedSQL when you want typed raw queries. Reach for these where you'd use EF's `FromSqlInterpolated` — window functions, CTEs, vendor-specific SQL.
- **Mental shift from EF Core** — no change tracking; there is no `SaveChanges()`. No `DbContext` scoped per request; one process-wide client. `update`/`upsert` are explicit `where` + `data` calls. No lazy loading; you opt in via `include`.

**Exercise:** model `User`, `Note`, `Tag` (M:N) in `schema.prisma`, run `migrate dev`, write: by-id query, with-tags-via-include, search-via-where. Add a transaction that atomically creates a Note + Tags. Trigger a drift situation (modify a column directly in psql) and see `migrate dev` complain.

---

### Phase 3 — Next.js App Router + RSC (4 days)

**The mental model first** — this is the only thing that *really* matters from Next.js:

- **RSC vs Client Components.** Default = Server. `"use client"` is a *boundary* — every import below it ships to the client. Props crossing must be React-serializable (no class instances, no functions except Server Actions; `Date` survives JSON; `Decimal`, `BigInt`, `Map` do NOT).
- **Children-passing across the boundary**: a Client Component can take Server children via `children` and keep server work on the server. Memorize this pattern.

**The decision tree — memorize this:**

- **Read data**
  - SSR + SEO + secret → **RSC `await fetch(hono)`** (forward auth on the server)
  - User-interactive (filters, debounced search) → **Client + TanStack Query** hitting Hono directly
  - Static-ish, shared across users → RSC + `'use cache'` + `cacheTag()`
- **Write data (you have Hono)**
  - **Default: call Hono directly.** Don't double-wrap.
  - Use a **Server Action** specifically when you want form ergonomics + same-origin cookies + same-RTT revalidation. The action itself `await`s Hono, then `revalidateTag(tag, profile)` or `updateTag(tag)`.
  - **Route Handler** for: webhooks, OAuth callbacks, BFF aggregation of multiple Hono calls, endpoints called by non-Next clients.
- **Protect a route**
  - Coarse redirect across many routes → `proxy.ts` with `export { auth as proxy }`
  - Section-wide gating + shared data → `layout.tsx` calls `auth()` + `redirect()`
  - Per-row authorization → `auth()` inside the page/RSC/Server Action.
  - **Proxy is NOT security.** Always re-check `auth()` and authorization inside every Server Action and data accessor. Matcher exclusions also skip Server Actions on those paths.

**Next.js 16 landmines:**

- `middleware.ts` → `proxy.ts` (codemod). Default runtime: `nodejs`. Edge stays in `middleware.ts`.
- Async dynamic APIs — `await params`, `await searchParams`, `await cookies()`, `await headers()`.
- **Cache Components** — opt in with `cacheComponents: true`. PPR removed-as-concept. Mark cached units with `'use cache'`, set `cacheLife(profile)`, tag with `cacheTag(tag)`. Invalidate with `revalidateTag(tag, profile)` (single-arg deprecated) or `updateTag(tag)` in Server Actions for read-your-writes.
- `fetch` not auto-cached since 15. GET Route Handlers dynamic by default since 15.

**Server Actions security — the briefs glossed this:**

A Server Action is a **public POST to its host route**. Next ships an **Origin header check + obfuscated action IDs**, NOT a CSRF token. It fails when:
- you enable `serverActions.allowedOrigins`,
- your reverse proxy strips/rewrites `Origin`/`Host`,
- a non-browser client calls the action.

So: re-check `await auth()` and authorization inside every action; rate-limit destructive actions; remember action IDs are stable across deploys and enumerable; verify your ingress preserves `Origin`/`Host`.

**Senior-React traps (the high-ROI list):**

- Reaching for `useEffect` to fetch on mount when an RSC `await` is the answer.
- Importing a file that reads `process.env.SECRET` into a Client Component — Next silently replaces non-`NEXT_PUBLIC_` vars with `""`. Use the `server-only` npm package — it crashes at build if a server-only module is bundled to the client. Mirror with `client-only`.
- **`NEXT_PUBLIC_` is inlined at build time.** Rotating a "public" key requires a rebuild.
- Wrapping the *root* layout in a Context Provider drags everything client-side. Push providers as deep as possible.
- Mutating without `revalidateTag`/`updateTag`/`router.refresh()` → stale UI.
- Hydration mismatches from `Date.now()`/`Math.random()`/locale formatting in RSC. Use `connection()` + Suspense or `'use cache'`.
- **`auth()` poisons static shells under `cacheComponents`** — any component that calls it becomes dynamic. Hoist or wrap with Suspense deliberately.
- **JSON serialization across RSC**: `Date` survives; `Decimal`, `BigInt`, `Map`, class instances do NOT. Use superjson or convert at the boundary.
- **Four similarly-named caches**: `React.cache()` (request memoization for fns), Next's `fetch` memoization (per-render), the Data Cache (`'use cache'`/`fetch` cache), the Router Cache (client-side navigation). Learn which lives where.

**Exercise:** build a Next.js app — `/notes` page (RSC reading via fetch to your Hono); `/notes/[id]/edit` with a Server Action that calls Hono and `revalidateTag`s; a Client Component interactive filter that calls Hono via TanStack Query; `proxy.ts` redirecting unauthenticated users from `/notes/*`; then re-check auth inside the Server Action and prove the proxy is not security.

---

### Phase 4 — Auth.js v5 (NextAuth) for your codebase (2 days)

> **2026 reality check** — Auth.js was handed over to the Better Auth team in September 2025. v5 never reached GA. The official site now hosts a "migrate to Better Auth" guide. Your team is on it, so you need it. But for any greenfield decision, advocate Better Auth or an external IdP (Clerk, WorkOS, Auth0, Keycloak, Authentik).
> Sources: [better-auth.com/blog/authjs-joins-better-auth](https://better-auth.com/blog/authjs-joins-better-auth) ; [authjs.dev/getting-started/migrate-to-better-auth](https://authjs.dev/getting-started/migrate-to-better-auth)

**Essentials:**

- One `auth.ts` at the project root calls `NextAuth({...})` and exports `{ handlers, auth, signIn, signOut }`. Mount `handlers` at `app/api/auth/[...nextauth]/route.ts`.
- `auth()` is the **universal server-side session reader** — replaces v4's `getServerSession`, `getToken`, `withAuth`. In a Client Component, wrap with `<SessionProvider>` and use `useSession`.
- **Edge config split**: a slim `auth.config.ts` (no DB / Node-only deps) for the edge-compatible portion + a full `auth.ts` for Node. The "two-file pattern" that trips everyone up.
- **Env vars**: `AUTH_*` prefix (v4 was `NEXTAUTH_*`). `AUTH_SECRET` is mandatory.
- **Sessions**: JWT (stateless, easy cross-service, no instant revocation, larger cookies) vs database via Prisma adapter (instant revocation, DB hit per request, smaller cookies). Credentials provider forces JWT. **Decide deliberately.** If compliance asks "can you revoke a token within 60s?" and you've picked JWT without a short TTL + denylist, you have a gap.
- **Prisma adapter schema**: `User`, `Account`, `Session`, `VerificationToken`. With JWT strategy the `Session` table is unused but the adapter still works.
- **Callbacks** (this is where mental models break):
  - `jwt({ token, user, account })` — runs on sign-in AND every session read. Stamp `token.userId`, `token.roles` here.
  - `session({ session, token | user })` — shapes what `auth()` returns. Copy from `token` to `session.user`.
  - **The #1 "my role is undefined" cause:** setting a field in `session` callback without first putting it on `token` in `jwt` callback.
  - `authorized({ request, auth })` — used by the proxy export to gate routes.
- **Reading the session in four contexts**:
  - RSC → `const session = await auth()`
  - Server Action → `const session = await auth()` at the top
  - Route Handler → `await auth()` inside, or `export const GET = auth((req) => {...})`
  - Client Component → `<SessionProvider>` + `useSession()`

**Critical gotcha — memorize:** Auth.js session cookies are encrypted **JWE (A256CBC-HS512)**, not signed JWTs. `hono/jwt` only handles signed JWS, so it cannot decode them. You will not "share the session cookie with Hono" via `hono/jwt`. See cross-service auth next.

---

### Phase 5 — Cross-service auth: Next.js ↔ Hono (2 days)

You have an Auth.js session in Next.js. Your Hono service needs to identify the user. Four options, in increasing order of seriousness:

1. **Forward the session cookie + decode with `@auth/core/jwt` on the Hono side** (sharing `AUTH_SECRET`). Works but couples Hono tightly to Auth.js's JWE format.

2. **Mint a separate Bearer token in Auth.js's `session` callback**, send `Authorization: Bearer` from Next to Hono, Hono verifies. **This is what most teams reach for.** Two flavors:
   - **HS256 + shared secret.** Cheap, single trust boundary. Key rotation is awful (both services in lockstep); no `aud`/`iss` separation; any service holding the secret can impersonate any user. *Acceptable for a single internal app — known cargo-cult for anything bigger.*
   - **RS256 / ES256 + JWKS.** Next is the issuer, exposes `/api/jwks`, Hono fetches+caches keys. Verify `iss`, `aud`, `exp`, `kid`. Rotation is `kid`-based. **This is the honest production default.**

3. **Full OAuth resource server** with Keycloak/Auth0/Authentik/WorkOS as issuer, NextAuth as a client, Hono as a resource server validating via JWKS. Right answer for multi-app/multi-team. Overkill for a single internal app.

4. **Better Auth** has first-class API-token issuance for backend services without JWE-gymnastics. Note for when migration is on the table.

**Where the token lives:**

- **Default: server-to-server only.** Your RSC/Server Action reads the Auth.js session (encrypted cookie), mints/loads an API token, calls Hono with `Authorization: Bearer`. The browser never sees the API token. No CSRF on the Next→Hono leg.
- **When that breaks** — and it will, eventually: SSE streaming from Hono to the browser, WebSockets, direct file uploads (don't proxy a 2 GB upload through Next), a future mobile/desktop client. Then you need: short-lived (5–15 min) browser-held access token, audience-restricted to Hono, refresh via a Next.js endpoint, never localStorage (httpOnly cookie or in-memory + refresh).

**CSRF / CORS:**

- Server-to-server (Next server → Hono): no browser cookies, no CSRF surface.
- Browser → Hono (cross-origin): `cors({ origin: 'https://app.example.com', credentials: true, allowMethods: [...], allowHeaders: ['Authorization', 'Content-Type'] })`. Wildcard origin is forbidden with credentials. If you authenticate via cookies, add a CSRF token (double-submit). If via Bearer, CSRF is moot but XSS becomes critical — short-lived tokens.
- **Server Actions**: Origin-checked (not token-CSRF). Verify your reverse proxy preserves `Origin`/`Host`.

**Exercise:** install JWT middleware in Hono (start with `@hono/jwt` HS256). Extend the Auth.js `jwt` callback to stamp `userId`+`roles` on `token`; in `session` callback, also mint an HS256 API token signed with a dedicated `API_JWT_SECRET` (NOT `AUTH_SECRET`). Call Hono from a Server Action server-side. Then graduate: build an RS256 + JWKS variant. As a third exercise, build an SSE endpoint on Hono and prove the server-to-server-only story doesn't work — design the short-lived browser-token escape hatch.

---

### Phase 6 — Production hardening (2 days)

- **Cache Components depth** — under `cacheComponents`, what makes a component dynamic (any `cookies()`/`headers()`/`auth()`/uncached fetch read). Build the static-shell-plus-Suspense-holes mental model. Debug `Uncached data accessed outside <Suspense>` errors.
- **`server-only` and `client-only` packages** — import-time guards that crash if a module ends up on the wrong side. Use them on anything touching secrets.
- **Floating promises** — enable `no-floating-promises` (typescript-eslint or Biome v2 nursery). `process.on('unhandledRejection', ...)` should crash, not swallow. Unlike C# (CS4014), Node won't warn you by default.
- **AbortController for cancellation** — pass `c.req.raw.signal` through to `fetch`/Prisma. `AbortSignal.timeout(5000)` for idiomatic timeouts.
- **Observability** — pino + `pino-opentelemetry-transport` + OTEL SDK auto-instrumentation. Propagate `traceparent` across the Next→Hono boundary. Child logger middleware that stamps `traceId` + `userId` on every log line.
- **Self-host realities** (if not on Vercel):
  - `output: 'standalone'` build
  - ISR cache handler if you use ISR (Vercel-only by default; pick a Redis-backed handler for self-host)
  - `sharp` installed for `next/image`
  - **Distroless + Prisma**: set `binaryTargets` (e.g. `linux-musl-openssl-3.0.x` for Alpine, `debian-openssl-3.0.x` for distroless), copy `node_modules/.prisma/client`. Easily two days the first time.
  - Native modules (`sharp`, `bcrypt`) need correct target triples.
- **Docker multi-stage** — builder runs `pnpm install --frozen-lockfile`, then `pnpm deploy --prod /out` to prune dev deps + flatten workspace links, copy `/out` into `gcr.io/distroless/nodejs24-debian12`. Non-root user. No runtime `npm install`, ever.
- **Process model** — container-per-CPU behind a load balancer (Kubernetes / Fly / Railway). Only reach for the `cluster` module on a single self-hosted VM where you can't horizontally scale containers.

---

### Phase 7 — Look up on demand (do NOT pre-study)

Know they exist; learn when you hit the problem:

- Parallel/intercepting routes, route groups
- `next/image`, `next/font`, the metadata API
- Streaming-from-server with Suspense
- React 19's `useFormStatus` / `useOptimistic` for action pending states
- TanStack Query (still valid alongside RSC for client-driven views)
- msw for HTTP mocking
- Changesets (only if you ship libraries)
- Workers / queues (BullMQ, Inngest, Trigger.dev) — what you reach for instead of `worker_threads`
- The OpenAPI generation story (`@hono/zod-openapi`) — only if your team committed to it
- TypedSQL in Prisma
- `require(esm)` interop quirks — read [Joyee Cheung's post](https://joyeecheung.github.io/blog/2025/12/30/require-esm-in-node-js-from-experiment-to-stability/) when CJS/ESM bites

---

## Reading list (curated, not survey)

**Read in order, once:**

1. Next.js: [Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components), [Layouts and Pages](https://nextjs.org/docs/app/getting-started/layouts-and-pages), [Caching](https://nextjs.org/docs/app/getting-started/caching), [Mutating Data](https://nextjs.org/docs/app/getting-started/mutating-data), [Proxy](https://nextjs.org/docs/app/api-reference/file-conventions/proxy), [Data Security](https://nextjs.org/docs/app/guides/data-security).
2. [Next.js 16 blog](https://nextjs.org/blog/next-16) and [Next.js 15 blog](https://nextjs.org/blog/next-15).
3. Auth.js: [v5 install](https://authjs.dev/getting-started/installation?framework=next.js), [migrating to v5](https://authjs.dev/getting-started/migrating-to-v5), [protecting resources](https://authjs.dev/getting-started/session-management/protecting), [Prisma adapter](https://authjs.dev/getting-started/adapters/prisma).
4. [The Better Auth handover post](https://better-auth.com/blog/authjs-joins-better-auth) — even if not migrating today.
5. Hono: [Best Practices](https://hono.dev/docs/guides/best-practices), [Validation](https://hono.dev/docs/guides/validation), [Testing](https://hono.dev/docs/guides/testing), [RPC](https://hono.dev/docs/guides/rpc), [Context](https://hono.dev/docs/api/context), [HTTPException](https://hono.dev/docs/api/exception).
6. Prisma: [Models](https://www.prisma.io/docs/orm/prisma-schema/data-model/models), [Relation queries](https://www.prisma.io/docs/orm/prisma-client/queries/relation-queries), [Migrate workflows](https://www.prisma.io/docs/orm/prisma-migrate/workflows/development-and-production), [Connection pool](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections/connection-pool), [Next.js dev practices](https://www.prisma.io/docs/orm/more/help-and-troubleshooting/help-articles/nextjs-prisma-client-dev-practices), [Transactions](https://www.prisma.io/docs/orm/prisma-client/queries/transactions).

**Save for when you hit the problem:**

- pino-opentelemetry-transport README (when wiring OTEL).
- PgBouncer transaction-mode caveats (when prepared-statement / advisory-lock errors appear).
- Lee Robinson's [Building APIs with Next.js](https://nextjs.org/blog/building-apis-with-nextjs) — explicit guidance on Server Actions vs Route Handlers vs external API for exactly your Hono setup.

---

## The single biggest assumption to challenge with your team

**Has the team chosen Auth.js v5 deliberately, or by inertia?** It's not GA, it was handed off to Better Auth in September 2025, and the official site now points to a migration guide. If this is a fresh codebase, raise it. If they already shipped, design the **cross-service token layer in Phase 5 so it's swappable** — a Better Auth migration should be a 2-week swap, not a rewrite. Concretely: keep your Hono JWT middleware verifying a generic JWT with `iss`/`aud`/`kid`; never let it depend on Auth.js types.

---

## Final shape — total time budget

| Phase | Topic | Days |
|---|---|---|
| 0 | Toolchain | 1 |
| 1 | Hono on Node end-to-end | 3 |
| 2 | Prisma | 2 |
| 3 | Next.js App Router + RSC | 4 |
| 4 | Auth.js v5 for this codebase | 2 |
| 5 | Cross-service auth (Next ↔ Hono) | 2 |
| 6 | Production hardening | 2 |
| **Total** | **~16 working days** (6–8 calendar weeks at a few hours/day) |

This is deliberately tighter than what the research briefs proposed. The cuts (PPR, worker_threads, event-loop phases, Buffer trivia, TS language refreshers, pnpm-as-topic, Biome-vs-ESLint debate) come from the senior-dev value review — they're week-1 demo content, not week-1 production prep.

---

## Appendix — Video & course track (vetted for currency, May 2026)

Every resource below was checked for a publication or update date recent enough to teach Next.js 15/16 + the Auth.js→Better Auth handover. Anything older than ~Sept 2024 was cut on principle.

### The free, authoritative track — do these first

1. **Vercel Academy — Next.js Foundations** — https://vercel.com/academy/nextjs-foundations
   The only structured course confirmed to teach the **Next.js 16 cache mental model** (`cacheComponents`, `'use cache'`, `cacheLife()`, `cacheTag()`, `revalidateTag`/`updateTag`). 26 lessons across Foundation → Core → Advanced → Polish. Fills Phase 3 and Phase 6.
2. **Next.js Learn — Dashboard App** — https://nextjs.org/learn/dashboard-app
   The canonical free Lee-Robinson-authored walkthrough that wires RSC fetches, Server Actions, `revalidateTag`, and NextAuth in one app. Fills Phase 3 + Phase 4. Pair with Vercel Academy on top because this still teaches pre-`cacheComponents` patterns.
3. **Next.js Conf 2025 — AMA with the Next.js team** (Lee Robinson, Delba de Oliveira, Jimmy Lai, Sebastian Markbåge, Josh Story, posted Oct 22 2025) — https://www.youtube.com/watch?v=UrilL6JII-U
   Authoritative source on `proxy.ts` rename, PPR-folded-into-Cache-Components, async dynamic APIs, straight from the people who shipped Next 16. Fills Phase 3 + Phase 6.
4. **Yusuke Wada — "Make Hono Work on Node.js"** (Node Congress 2025) — https://gitnation.com/contents/make-hono-work-on-nodejs
   The Hono creator's own explanation of `@hono/node-server`'s Fetch-to-Node bridging. Fills Phase 1. HonoConf 2025 talks at https://honoconf.dev/2025 add useful ecosystem context.
5. **devtools.fm Ep. 150 — Bereket Engida (Better Auth maintainer)** — https://www.devtools.fm/episode/150
   API-token-issuance and Auth.js→Better Auth migration framing direct from the maintainer. Long-form interview also at https://www.youtube.com/watch?v=OpVeRCArKts (Aug 2025). Fills Phase 4 + Phase 5.

### The paid track — only what's worth the money

1. **Frontend Masters — Next.js Fundamentals v4** (Scott Moss, published Apr 3 2025, 6h 41m) — https://frontendmasters.com/courses/next-js-v4/
   Covers App Router + Server Actions + caching + auth from someone who teaches what he uses in production. **Caveat:** April 2025 predates Next 16's `proxy.ts` rename and the Cache Components reshuffle, so treat the cache sections as Next 15 mental model and **overlay Vercel Academy on top.** Best Phase 3 resource available paid.
2. **Frontend Masters — React & TypeScript v3** (Steve Kinney) — https://frontendmasters.com/courses/react-typescript-v3/
   Only worth it if your TS isn't already razor-sharp — covers discriminated unions for reducer typing, reusable generics, and Zod schema patterns you'll reuse in Hono validators. Skip if you're confident.

**One Frontend Masters month ($39) covers both. That's the only paid spend recommended.**

### YouTube — specific videos worth bookmarking (not "subscribe to the channel")

- **Lee Robinson — "Self-Hosting Next.js"** (Oct 2024) — https://www.youtube.com/watch?v=sIVL4JMqRfc and repo https://github.com/leerob/next-self-host
  The only authoritative walkthrough of Docker + Postgres + Nginx + `output: 'standalone'` + Prisma binary targets in one place. Fills Phase 6. Deployment mechanics unchanged despite borderline date.
- **Jack Herrington — "Partial Page Caching Using React Server Components"** — https://www.youtube.com/watch?v=t9xB8xvySyo
  Deep dive on the static-shell-plus-Suspense-holes mental model. Pair with Vercel Academy's Cache Components lesson.

### Deliberately NOT recommended

- Udemy "Complete Guide" / "Bootcamp" courses (Maximilian Schwarzmüller etc.) — perpetual churn, mix Pages Router with App Router.
- 14-hour "Next.js 15 Full Course" anonymous YouTube monoliths — beginner content padded out.
- Code with Antonio SaaS-clone build-alongs — production value is high, but you'd be typing along, not learning patterns. Wrong format for a senior.
- Frontend Masters Intermediate Next.js (Scott Moss, July 2024) — pre-dates Next 15's "fetch no longer auto-cached" default. Teaches wrong defaults. Use v4 instead.
- Brian Holt's Complete Intro to React v9 — beginner React. Skip.
- Any pre-Sept-2025 Auth.js v5 tutorial — teaches a frozen target. Auth.js was handed to Better Auth in Sept 2025.
- Prisma Day recordings 2022–2023 — pre-RSC, pre-`relationLoadStrategy`. The official Prisma + Next.js docs (already in your reading list) are more current.

### Per-phase course mapping

| Phase | Best resource(s) |
|---|---|
| 0 — Toolchain | Skip courses. Optional: Steve Kinney's **React & TypeScript v3** if TS needs sharpening. |
| 1 — Hono on Node | **Yusuke Wada's "Make Hono Work on Node.js"** + Hono official docs. HonoConf 2025 for ecosystem context. |
| 2 — Prisma | Official Prisma docs are more current than any video. No course recommended — Phase 2 is doing, not watching. |
| 3 — Next.js App Router + RSC | **Vercel Academy Next.js Foundations** (free, Next 16) + **Next.js Learn Dashboard App** (free) + **Scott Moss Next.js v4** (paid) + **Next.js Conf 2025 AMA**. |
| 4 — Auth.js v5 | Auth.js official v5 install + migration docs. **devtools.fm Ep. 150 with Bereket Engida** for handover framing. Next.js Learn dashboard chapter on NextAuth wiring. |
| 5 — Cross-service auth | No single video covers this well — your roadmap's own JWE/JWS analysis is the most current source. Pair with **Lee Robinson's "Building APIs with Next.js"** blog and **Bereket Engida** for Better Auth's API-token escape hatch. |
| 6 — Production hardening | **Lee Robinson "Self-Hosting Next.js"** video + repo + **Jack Herrington "Partial Page Caching"** for Cache Components debugging. |

**The brutal summary:** Vercel Academy + Next.js Learn + the Conf 2025 AMA + Scott Moss v4 + Lee Robinson's self-host video + the Bereket Engida interview = the entire curriculum. One Frontend Masters month covers the paid sliver. Everything else is noise.
