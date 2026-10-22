# Forge — backend & infra conventions

**Audience:** an AI coding agent (or human) building a new internal tool for team 6324. This is the backend/infra half of the standard; see `ui-standards.md` in this same folder for the frontend/visual half.

The canonical, living implementation of everything here is the `frc-attendance` app. When in doubt, read that repo's actual code — this file explains the pattern, the code is the proof it works.

## 1. The stack, and why

| Layer | Choice | Why |
|---|---|---|
| Framework | Next.js (App Router), TypeScript | One deploy target for frontend + API + cron, server components by default |
| Hosting | Vercel (Hobby tier is fine for a team-sized tool) | Zero-ops, free tier, git-push deploys, built-in Cron |
| Database | Neon Postgres (serverless) | Free tier, scales to zero, HTTP driver works great in serverless functions (no connection pooling headaches) |
| ORM / migrations | Drizzle ORM + drizzle-kit | Typed schema is the single source of truth; migrations are plain generated SQL you can read and review |
| Auth | Auth.js (next-auth v5), one OAuth provider (usually Google), JWT sessions | No session table, no adapter, works great on serverless. Role/active status are re-read from the DB on every request — never trust the JWT for anything but identity |
| Styling | Tailwind v4 | Design tokens become real CSS custom properties — see `ui-standards.md` §2 |
| Local dev DB | PGlite (`pglite://` URL) | Zero-install embedded Postgres; same schema/migrations as production, no Docker required for a quick start |
| Tests | Vitest + PGlite | Integration tests run migrations against a real (embedded) Postgres, not mocks |

Don't substitute pieces of this casually (e.g. swapping in Prisma, or a different hosting provider) without a real reason — the value of this document is that every 6324 tool works the same way, so anyone (or any agent) can pick up any of them.

## 2. Repo layout

```
src/
  app/                  # Next.js App Router: pages, route handlers, server actions
    api/                # route.ts handlers (webhooks, cron targets, polling endpoints)
    <feature>/
      page.tsx
      actions.ts        # "use server" mutations for that feature
  components/           # shared UI (see ui-standards.md)
  db/
    schema.ts           # Drizzle schema — the single source of truth
    index.ts            # lazy DB client (see §4)
    types.ts            # AnyDb driver-agnostic type
  lib/                  # everything else: auth, authz, business logic, pure helpers
drizzle/                # generated migrations (0000_*.sql, 0001_*.sql, ...) + meta/
tests/                  # vitest, one file per lib module roughly
scripts/
  seed.ts               # dev-only seed data (never run against production)
.env.example            # every env var, documented, with instructions on where to get it
vercel.json             # cron jobs only, if any
drizzle.config.ts
AGENTS.md               # see §8
```

## 3. Schema and migrations

- `src/db/schema.ts` is the only place tables are defined. Every table gets a corresponding `export type X = typeof x.$inferSelect`.
- Use `pgEnum` for closed sets of string values (roles, status/type fields) instead of a plain `text` column — it gives you a real Postgres enum, a typed `.enumValues` tuple you can feed straight into `z.enum()`, and a compile error if you typo a value anywhere.
- Singleton/settings-style tables (site-wide config) use a fixed-string primary key (`id: text("id").primaryKey().default("global")`) rather than inventing a separate "is there a row yet" check everywhere.
- Partial unique indexes (`uniqueIndex(...).on(t.isOpen).where(sql\`${t.isOpen} = true\`)`) are the idiomatic way to enforce "at most one X in state Y" — e.g. at most one open meeting — without application-level locking.
- **Never hand-write a migration from scratch.** Change `schema.ts`, then run `npm run db:generate`, then open the generated SQL and sanity-check it. Rename the generated file to something descriptive (`0004_add_meeting_type.sql` instead of the random `0004_powerful_ender_wiggin.sql` drizzle-kit invents) and update the matching `tag` in `drizzle/meta/_journal.json` to match — keep those two in sync or `drizzle-kit migrate` gets confused.
- If a migration needs to seed a default row (e.g. the one settings row), add a plain `INSERT ... ON CONFLICT DO NOTHING;` statement to the generated SQL file. That's the one case it's fine to hand-edit a generated migration.
- `drizzle.config.ts` prefers `DATABASE_URL_UNPOOLED` over `DATABASE_URL` for migrations (migrations want a direct connection, not Neon's pooled one), and switches to the `pglite` driver automatically when the URL starts with `pglite://`. Copy this file's logic verbatim into a new project.

## 4. Database client

Lazy-initialize the DB client behind a `Proxy` so importing `@/db` never connects — this is what lets `next build` succeed with zero environment variables set (no secrets needed just to compile):

```ts
export const db: DB = new Proxy({} as DB, {
  get(_t, prop) {
    const real = getDb();
    const v = (real as any)[prop];
    return typeof v === "function" ? v.bind(real) : v;
  },
});
```

`getDb()` branches on the `DATABASE_URL` scheme: `neon(url)` + `drizzle-orm/neon-http` in production, PGlite in dev when the URL is `pglite://...`. Never use a pooling driver (`pg`, `postgres.js` with a persistent pool) for the production path — Vercel functions are short-lived and a connection pool doesn't survive between invocations; Neon's HTTP driver is stateless per-request, which is exactly what you want.

## 5. Config / environment variables

One `src/lib/env.ts`, all getters, nothing read directly via `process.env` elsewhere in the app (except `layout.tsx`/`nav.tsx` for genuinely static display values like a team name, which is fine). Pattern:

```ts
export const env = {
  get databaseUrl(): string {
    const url = process.env.DATABASE_URL;
    if (!url) throw new Error("DATABASE_URL is not set");
    return url;
  },
  // ...
  get appUrl(): string | null {
    const explicit = process.env.APP_URL?.trim();
    if (explicit) return explicit.replace(/\/+$/, "");
    const vercel = process.env.VERCEL_PROJECT_PRODUCTION_URL?.trim();
    if (vercel) return `https://${vercel}`;
    return null;
  },
};
```

Rules:
- Required secrets throw a clear error *when read*, not at module load — this is what keeps the build working without secrets.
- Everything optional has a sane default baked into the getter, not scattered `?? "fallback"` calls at every call site.
- Fall back to Vercel's own injected vars (`VERCEL_PROJECT_PRODUCTION_URL`, etc.) before making the user set something Vercel already knows.
- `.env.example` documents every single var: what it's for, whether it's required, and *exactly* where to get its value (which console, which menu). A future agent (or human) should never have to guess.

## 6. Auth and authorization

- One OAuth provider is enough for an internal team tool — don't build a credentials/password flow.
- JWT session strategy, no database adapter, no session table. The JWT carries only the user's internal DB id (`token.uid`).
- **Role and active-status checks always re-read the database on every request.** Put this in one memoized (`React cache()`) `getCurrentUser()` and never branch on anything from the JWT except identity. This is what makes role changes and deactivating someone take effect immediately instead of waiting for a token to expire.
- A `requirePageUser(minRole, callbackPath)` helper for pages (redirects to sign-in or a "forbidden" page) and a `requireRole(minRole)` helper for server actions/route handlers (throws a typed `AuthzError` that a shared `guard()` wrapper turns into a friendly message) covers nearly every authorization need. Define a numeric `RANK` map for roles so "at least mentor" is a one-line comparison.
- Gate who's allowed to sign in at all (email domain allowlist, an explicit admin-emails bootstrap list, "roster only") in one `decideSignIn()` pure function that's unit-testable without touching Auth.js or a database.

## 7. Server actions and mutations

- Every mutation is a `"use server"` action with the signature `(prev: ActionState, fd: FormData) => Promise<ActionState>`, where `ActionState = { ok: boolean; message: string } | null`.
- Wrap the body in a shared `guard()` helper that catches `AuthzError` and `z.ZodError` and turns them into a friendly `fail(message)` instead of a crash — reserve the generic "Something went wrong" message for truly unexpected errors, log those server-side.
- Parse `FormData` with `zod`, always — never trust a hidden field's value without validating it server-side (ids should be `z.uuid()`, enums should be `z.enum(someTable.someEnum.enumValues)`).
- After any mutation that affects more than one page, just `revalidatePath("/", "layout")` — at the scale of an internal tool, fine-grained revalidation isn't worth the complexity.
- Pair every action with an `<ActionForm>` client component (`useActionState` + an `aria-live` result message, optional native `confirm()` gate for destructive actions, optional reset-on-success) — see `ui-standards.md` §5. Never build a bespoke fetch-and-setState mutation flow when this pattern covers it.

## 8. The `AGENTS.md` trick

Put an `AGENTS.md` at the repo root (and Next.js will also auto-generate one inside `node_modules/next/AGENTS.md` on `next dev` — don't fight that, just keep it committed) telling any agent working in the repo: **read the actual installed framework's docs before writing code that touches an API you're not 100% sure of**, especially after a major version bump. Concretely:

```markdown
## This is NOT the Next.js you know

Next.js, React, and other fast-moving dependencies change real APIs between
major versions. Read the relevant guide in `node_modules/next/dist/docs/`
(resolved from this file's directory) before writing code that touches
routing, metadata/viewport, caching, or server actions. Heed deprecation
notices.
```

This isn't boilerplate — it has caught real behavior changes (e.g. `generateViewport` replacing a static export once a layout needs request-time data, or `cacheComponents` changing when a dynamic value is allowed to be read) that would otherwise have been "fixed" by writing code that matches an older, memorized API. When something about the framework's behavior surprises you, the fix is to go read the docs that shipped with the actual installed version, not to guess or fall back on training data.

## 9. Testing

- Vitest, `tests/**/*.test.ts`, one file per `lib` module, direct function calls — no HTTP layer in the test loop.
- For anything that touches the database, spin up a **real** (embedded) Postgres per test file: `new PGlite()` (in-memory, no path = ephemeral), replay every file in `drizzle/*.sql` in order split on `--> statement-breakpoint`, wrap with `drizzle({ client, schema })`. This exercises your actual migrations, not a hand-maintained schema mirror that can drift.
- A module that starts with `import "server-only"` will throw if a test imports it directly outside a Server Component context. Stub it per-test-file with `vi.mock("server-only", () => ({}))` followed by a dynamic `await import(...)` of the real module — don't remove `"server-only"` from the source file just to make it testable.
- `npm run typecheck && npm run lint && npm test` all green is the bar before calling anything done, every time — not just before a PR.

## 10. Deploying

See the `deploy-6324-app` skill in this plugin for the actual step-by-step with CLI commands and confirmation checkpoints. The short version:

1. GitHub repo exists, Vercel project is linked to it.
2. Neon project + database exists; migrations have been run against it with `DATABASE_URL_UNPOOLED` pointed at Neon's *direct* (non-pooler) connection string.
3. Vercel env vars are set for Production (and ideally Preview too): the pooled `DATABASE_URL`, `AUTH_SECRET`, the OAuth client id/secret, any app-specific secret, `ADMIN_EMAILS` so someone becomes an admin on first sign-in.
4. OAuth provider's console has both the production domain and any preview/localhost origins registered as authorized origins/redirects.
5. If the app has a `vercel.json` cron, double check the schedule is in UTC and matches the app's actual team-timezone-aware logic — cron schedules themselves can't be timezone-aware.
