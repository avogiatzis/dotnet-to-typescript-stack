# dotnet-to-typescript-stack

A migration roadmap for a senior C# / .NET + functional React/TS developer joining a **Next.js (App Router) + NextAuth/Auth.js v5 + Hono-on-node-server + Prisma** project.

See [`ROADMAP.md`](./ROADMAP.md) for the full document.

## What's in it

- **8 phases**, ~16 working days (6–8 calendar weeks at a few hours/day)
- Concept-by-concept ASP.NET Core ↔ Hono and EF Core ↔ Prisma mappings
- Decision trees for: read data, write data, protect a route, cross-service auth
- Senior-React traps + Server Action security gotchas + PgBouncer footguns
- An appendix with vetted 2026 video/course recommendations
- A 2026 ecosystem reality check (Next.js 16, Auth.js → Better Auth handover, Node 24 native TS)

## How it was built

Four parallel research agents (Next.js gaps, Node/Hono for ASP.NET Core devs, Prisma + Auth.js + cross-service auth, TS/Node ecosystem) → two adversarial reviewers (senior-dev value + technical skepticism) → one realism + source validator → synthesis. The review cycle killed entire topics (PPR, `worker_threads`, event-loop phase tours) and caught the Auth.js → Better Auth handover that a solo researcher would have missed.

## License

CC0 — public domain. Fork, copy, edit.
