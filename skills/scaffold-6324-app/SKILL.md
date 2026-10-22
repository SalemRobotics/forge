---
name: scaffold-6324-app
description: Scaffold a brand-new internal tool for team 6324 using the team's standard stack (Next.js App Router + TypeScript + Tailwind v4 + Drizzle + Neon + Auth.js, deployed to Vercel) and the team's visual design system. Use when the user wants to start a new app/tool/dashboard for the team, asks to "build it like the attendance app", or explicitly invokes this skill. This only sets up local files — it does not touch git remotes, Vercel, or Neon; use the deploy-6324-app skill for that.
---

# Scaffold a new 6324 app

This produces a new local Next.js project that looks and behaves like every
other 6324 tool, so a future agent (or human) can pick it up without
relearning conventions. It does **not** create any cloud resources or push
anywhere — that's the separate `deploy-6324-app` skill, deliberately, because
scaffolding is fully reversible (it's just local files) and deploying is not.

## Step 0 — read the standards first

Before writing anything, read both reference docs that ship with this
plugin:

- `references/ui-standards.md` — the visual/component system (colors,
  typography, the component catalog, accessibility rules).
- `references/stack-conventions.md` — the backend/infra conventions (repo
  layout, schema/migration rules, auth pattern, server-action pattern,
  testing pattern).

Everything below assumes you've read both. Don't improvise a different
pattern for something those docs already cover.

## Step 1 — ask what's actually different about this tool

Don't re-derive this from scratch every time — ask the user (or infer from
their request) just the handful of things that vary per app:

- App name / one-line purpose (becomes `TEAM_NAME`-equivalent branding and
  the repo name).
- What roles exist beyond "everyone who signs in" — does it need the same
  student/mentor/admin three-tier roster, or something simpler (e.g. just
  "signed in" vs "admin")?
  Keep it as close to the attendance app's `role` enum + `RANK` pattern as
  the tool actually needs — don't build a permissions system more elaborate
  than the tool requires.
- Does it need a cron job (like the attendance app's 9pm auto-checkout)? If
  so, what schedule, and what does it do?
- Brand hue: reuse one of the documented palettes in `ui-standards.md` §3.1
  (navy/teal/green/purple/red/orange) unless the user wants something new,
  in which case generate a new 11-step scale following that section's rules
  and contrast constraints.

## Step 2 — scaffold

1. `npx create-next-app@latest <name> --typescript --tailwind --app --eslint --src-dir --no-import-alias` — then immediately fix the import alias to `@/*` in `tsconfig.json` to match convention (the flag above avoids create-next-app fighting you over it; just set it by hand afterward).
2. Install the stack's dependencies (match the versions in `stack-conventions.md` §1 / the attendance app's `package.json` unless there's a specific reason to diverge):
   ```
   npm install @neondatabase/serverless drizzle-orm lucide-react next-auth@beta qrcode zod server-only swr
   npm install -D @electric-sql/pglite drizzle-kit dotenv tsx vitest postgres
   ```
   (Drop `qrcode`/`swr` if this tool has no kiosk-style live-polling screen.)
3. Copy, don't reinvent, these files from the attendance app (adjust only the app-specific bits called out):
   - `src/db/index.ts`, `src/db/types.ts` — verbatim.
   - `drizzle.config.ts` — verbatim.
   - `src/lib/env.ts` — keep the pattern, trim to the env vars this app actually needs.
   - `src/lib/actions.ts` — verbatim (`guard`, `ok`, `fail`, `formObject`, `revalidateAll`).
   - `src/lib/auth.ts`, `src/lib/authz.ts` — keep the pattern; simplify the role enum if this tool doesn't need three tiers.
   - `src/app/globals.css` — copy the `@theme` block and token structure from `ui-standards.md` §3, substitute the chosen brand hue.
   - `src/components/ui.tsx`, `button-class.ts`, `action-form.tsx`, `nav-bar.tsx`, `nav.tsx` — copy, then delete whatever domain-specific pieces don't apply (e.g. this tool probably has no `TargetProgressBar`).
   - `.env.example`, `AGENTS.md` (see `stack-conventions.md` §8), `vitest.config.mts`, `tests/` harness pattern (`freshDb()` from any `*.db.test.ts` file).
4. Write `src/db/schema.ts` for this app's actual domain, following `stack-conventions.md` §3 (enums for closed sets, partial unique indexes for "at most one X", `$inferSelect` types for everything).
5. `npm run db:generate` to produce the first migration, rename it from the random drizzle-kit name to something descriptive, and fix the matching tag in `drizzle/meta/_journal.json`.
6. Set `DATABASE_URL="pglite://.data/pglite"` in a local `.env.local` and run `npm run db:migrate` to confirm the schema actually applies before building anything on top of it.
7. Build the first page: app shell (header/nav per `ui-standards.md` §4.1) + one real `PageHeader` + `Card` + at least one `EmptyState`, per that doc's §10 bootstrapping checklist — the shell should look finished before any real data flows through it.
8. `npm run typecheck && npm run lint && npm test` — all green before telling the user scaffolding is done.

## Step 3 — hand off

Tell the user scaffolding is complete and local-only: nothing has been
pushed, no Vercel project or Neon database exists yet. Point them at the
`deploy-6324-app` skill when they're ready to go live, and remind them that
skill will pause for their confirmation before anything that touches a
shared remote, a live deploy, or a billed cloud resource.
