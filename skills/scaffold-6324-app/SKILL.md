---
name: scaffold-6324-app
description: Scaffold a brand-new internal tool for team 6324 — any domain (attendance, inventory, scouting, scheduling, whatever's next) — using the team's standard stack (Next.js App Router + TypeScript + Tailwind v4 + Drizzle + Neon + Auth.js, deployed to Vercel) and the team's visual design system. Use whenever the user wants to start a new app/tool/dashboard for the team, or explicitly invokes this skill. This only sets up local files — it does not touch git remotes, Vercel, or Neon; use the deploy-6324-app skill for that.
---

# Scaffold a new 6324 app

This produces a new local Next.js project that looks and behaves like every
other 6324 tool, regardless of what the tool actually does — the stack and
conventions are domain-agnostic. A future agent (or human) should be able to
pick up any app built this way without relearning anything. It does **not**
create any cloud resources or push anywhere — that's the separate
`deploy-6324-app` skill, deliberately, because scaffolding is fully
reversible (it's just local files) and deploying is not.

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
their request) just the handful of things that vary per app. These
questions are deliberately generic; answer them fresh for whatever this
tool actually is, don't default to any other 6324 app's answers:

- App name / one-line purpose (becomes the branding and the repo name).
- What's the core domain model? (The thing(s) the app is actually about —
  equipment, events, shifts, parts, whatever. This drives `schema.ts` in
  Step 2.4.)
- What roles does it need, if any, beyond "signed in" vs "not"? Some tools
  need nothing more than that. Some need an admin tier. Some need three or
  more tiers with different capabilities. Size the `role` enum + `RANK`
  pattern (see `stack-conventions.md` §6) to exactly what *this* tool needs
  — don't build a permissions system more elaborate than the request.
- Does it need any scheduled/background job (a Vercel Cron hitting a route
  on a timer)? If so, what should run, and on what schedule?
- Does it need a live-updating/polling screen anywhere (a client component
  that polls a small API route every few seconds — e.g. a shared-display
  dashboard, a kiosk, a status board), or is plain request/response enough?
  Most tools don't need this; only pull in a polling dependency like `swr`
  if something in the request actually calls for a live display.
- Brand hue: reuse one of the documented palettes in `ui-standards.md` §3.1
  (navy/teal/green/purple/red/orange) unless the user wants something new,
  in which case generate a new 11-step scale following that section's rules
  and contrast constraints.

## Step 2 — scaffold

1. `npx create-next-app@latest <name> --typescript --tailwind --app --eslint --src-dir --no-import-alias` — then immediately fix the import alias to `@/*` in `tsconfig.json` to match convention (the flag above avoids create-next-app fighting you over it; just set it by hand afterward).
2. Install the stack's dependencies (versions per `stack-conventions.md` §1):
   ```
   npm install @neondatabase/serverless drizzle-orm lucide-react next-auth@beta zod server-only
   npm install -D @electric-sql/pglite drizzle-kit dotenv tsx vitest postgres
   ```
   Only add `qrcode`/`swr` if this specific app needs a live-polling display
   screen (per Step 1) — most tools don't.
3. These four files are genuinely domain-agnostic — the same for every
   6324-stack app regardless of what it does — so copy them verbatim from
   this plugin's `templates/` directory instead of writing them from
   scratch:
   - `templates/db/index.ts` → `src/db/index.ts`
   - `templates/db/types.ts` → `src/db/types.ts`
   - `templates/drizzle.config.ts` → `drizzle.config.ts`
   - `templates/lib/actions.ts` → `src/lib/actions.ts`
4. Everything else below is a *pattern* to follow, not a file to copy — write
   it fresh for this app, matching the shape described in the reference
   docs:
   - `src/lib/env.ts` — the getter pattern from `stack-conventions.md` §5,
     containing only the env vars this specific app actually needs.
   - `src/lib/auth.ts`, `src/lib/authz.ts` — the JWT-session, re-read-role-
     every-request pattern from `stack-conventions.md` §6, with a `role`
     enum and `RANK` map sized to what this app's Step 1 answers called for
     (which may be nothing more than "signed in or not").
   - `src/app/globals.css` — the `@theme` block and token structure from
     `ui-standards.md` §3, with this app's chosen brand hue substituted in.
   - `src/components/ui.tsx`, `button-class.ts`, `action-form.tsx`,
     `nav-bar.tsx`, `nav.tsx` — the component catalog in `ui-standards.md`
     §5. Build only the components this app actually uses; don't port over
     something domain-specific to a different tool (a progress bar, a kiosk
     QR display, etc.) just because it exists elsewhere.
   - `.env.example`, `AGENTS.md` (template text in `stack-conventions.md`
     §8), `vitest.config.mts`, and the `freshDb()` test-harness pattern
     (`stack-conventions.md` §9) — all short enough to write directly from
     those sections.
5. Write `src/db/schema.ts` for this app's actual domain (from Step 1's
   answers), following `stack-conventions.md` §3 (enums for closed sets,
   partial unique indexes for "at most one X", `$inferSelect` types for
   everything).
6. `npm run db:generate` to produce the first migration, rename it from the
   random drizzle-kit name to something descriptive, and fix the matching
   tag in `drizzle/meta/_journal.json`.
7. Set `DATABASE_URL="pglite://.data/pglite"` in a local `.env.local` and run
   `npm run db:migrate` to confirm the schema actually applies before
   building anything on top of it.
8. Build the first page: app shell (header/nav per `ui-standards.md` §4.1) +
   one real `PageHeader` + `Card` + at least one `EmptyState`, per that
   doc's §10 bootstrapping checklist — the shell should look finished before
   any real data flows through it.
9. `npm run typecheck && npm run lint && npm test` — all green before
   telling the user scaffolding is done.

## Step 3 — hand off

Tell the user scaffolding is complete and local-only: nothing has been
pushed, no Vercel project or Neon database exists yet. Point them at the
`deploy-6324-app` skill when they're ready to go live, and remind them that
skill will pause for their confirmation before anything that touches a
shared remote, a live deploy, or a billed cloud resource.
