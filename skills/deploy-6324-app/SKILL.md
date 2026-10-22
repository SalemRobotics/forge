---
name: deploy-6324-app
description: Deploy a 6324-stack app (Next.js + Drizzle + Neon + Vercel, see stack-conventions.md) to production using the Vercel CLI, Neon CLI, and git — installing those CLIs if missing. Use when the user asks to deploy, ship, or go live with one of these apps, connect it to Vercel/Neon, or set up hosting. Pauses for explicit user confirmation before anything hard to reverse (first push to a shared remote, first production deploy, creating a billed cloud resource); everything else (installing CLIs, local git init, linking, running migrations against a database the user already approved) proceeds without asking.
---

# Deploy a 6324-stack app

This automates the deploy walkthrough in `stack-conventions.md` §10 using
the Vercel CLI, the Neon CLI (`neonctl`), and `git` — so it doesn't need to
be done by hand every time. It is **not** fully unattended: three specific
moments always pause for the user's explicit go-ahead, because they're the
moments that are hard or impossible to undo, or that cost money, or that
become visible to other people. Everything else runs straight through.

**Do not silently skip the three checkpoints below, and do not reinterpret
"confirm once per major step" as "ask every single time" either** — ask
once per checkpoint per deploy session, then proceed through any repeated
instances of that same step (e.g. redeploying again later in the same
conversation) without re-asking.

## The three checkpoints (always stop and ask)

1. **First `git push` to a shared remote** (creating the GitHub repo and/or
   pushing to it for the first time). State plainly what will become
   visible and to whom (public vs private repo, under which account/org).
2. **Creating a new Neon project/database** (or any other billed resource).
   State what tier/plan this lands on and what it costs (Neon's free tier
   covers a team-sized tool; say so explicitly if that's the plan).
3. **The first `vercel --prod` deploy** (making the app live at a public
   URL). State the URL it will be live at.

Routine, already-approved follow-ups — redeploying after a code change,
running a migration against a Neon database that's already provisioned and
approved, pushing a subsequent commit to a repo that's already public — do
not need to re-ask.

## Step 0 — check for the CLIs, install what's missing

No confirmation needed for this step; installing a local CLI tool is fully
reversible and touches nothing shared.

```bash
git --version || echo "MISSING: git"           # almost always already present
vercel --version || npm install -g vercel       # Vercel CLI
neonctl --version || npm install -g neonctl     # Neon CLI
gh --version || echo "no gh CLI — fall back to manual GitHub repo creation"
```

`gh` (GitHub CLI) is optional — if it's present and authenticated
(`gh auth status`), use it to create the GitHub repo non-interactively.
If it's missing or not authenticated, skip straight to asking the user for
either a remote URL to push to, or permission to create one via `gh auth
login` (which itself requires the user to complete a browser/device flow —
walk them through it, don't try to bypass it).

## Step 1 — check for credentials; if missing, stop and explain, don't obtain them yourself

Non-interactive Vercel/Neon CLI use needs API tokens as environment
variables. **Never try to generate, guess, or interactively log in on the
user's behalf to obtain these** — if they're not already set, tell the user
exactly where to get them and wait.

```bash
test -n "$VERCEL_TOKEN" || echo "Missing VERCEL_TOKEN"
test -n "$NEON_API_KEY" || echo "Missing NEON_API_KEY"
```

If either is missing, tell the user:

- **VERCEL_TOKEN**: Vercel dashboard → Account Settings → Tokens →
  Create Token. `export VERCEL_TOKEN=...` before continuing. (Alternative:
  run `vercel login` interactively instead of using a token — that works
  too, it just requires them to complete it in a browser themselves.)
- **NEON_API_KEY**: Neon console → Account Settings → API Keys →
  Create API Key. `export NEON_API_KEY=...` before continuing.

Then stop and wait for the user to come back with those set, rather than
proceeding with half the pipeline non-functional.

## Step 2 — GitHub repo (checkpoint 1)

1. If no git repo yet: `git init`, write `.gitignore` per
   `stack-conventions.md`, make the first commit.
2. **Ask the user before creating or pushing to any remote.** Tell them the
   proposed repo name and visibility (default to **private** unless they
   say otherwise — this is an internal team tool).
3. On confirmation: `gh repo create <org-or-user>/<name> --private --source=. --remote=origin` (or the equivalent manual flow if `gh` isn't available: create the repo in the GitHub UI, then `git remote add origin <url>`), then `git push -u origin main`.

## Step 3 — Neon database (checkpoint 2)

1. **Ask the user before creating the Neon project.** Tell them it's free
   tier unless they've specified otherwise.
2. On confirmation:
   ```bash
   neonctl projects create --name <app-name> --api-key "$NEON_API_KEY"
   neonctl connection-string --project-id <id> --api-key "$NEON_API_KEY"               # pooled — becomes DATABASE_URL
   neonctl connection-string --project-id <id> --api-key "$NEON_API_KEY" --pooled=false # direct — becomes DATABASE_URL_UNPOOLED
   ```
3. Run migrations against the **direct/unpooled** URL, same as local dev:
   ```bash
   DATABASE_URL_UNPOOLED="<direct-url>" npm run db:migrate
   ```
   This does *not* need a fresh confirmation — the user already approved
   creating this database in step 1 of this section, and running the
   project's own reviewed migrations against a database that only this
   deploy just created is a routine, expected follow-through, not a new
   irreversible action.

## Step 4 — Vercel project + environment variables

No new checkpoint here — linking a project and setting env vars doesn't go
live or touch anyone else until the deploy in Step 5.

```bash
vercel link --yes --token="$VERCEL_TOKEN"
vercel env add DATABASE_URL production --token="$VERCEL_TOKEN"            # paste the pooled Neon URL
vercel env add AUTH_SECRET production --token="$VERCEL_TOKEN"             # openssl rand -base64 33
vercel env add AUTH_GOOGLE_ID production --token="$VERCEL_TOKEN"
vercel env add AUTH_GOOGLE_SECRET production --token="$VERCEL_TOKEN"
vercel env add ADMIN_EMAILS production --token="$VERCEL_TOKEN"
# ...plus whatever app-specific vars this tool's .env.example documents.
```

`vercel env add` prompts for the value interactively per variable (it
deliberately never takes secrets as plain CLI arguments, so they don't end
up in shell history) — walk the user through pasting each one rather than
trying to script around that.

Remind the user at this point, same as the manual walkthrough: if the app
uses Google OAuth, its console needs the Vercel production URL (and
whatever custom domain comes later) added as an authorized origin and
redirect URI before sign-in will work.

## Step 5 — first production deploy (checkpoint 3)

**Ask the user before this step**, stating the URL that's about to go
live (`vercel link` output / `vercel project ls` shows the assigned
`*.vercel.app` domain before you deploy).

On confirmation:
```bash
vercel --prod --token="$VERCEL_TOKEN" --yes
```

## Step 6 — verify

- Hit the deployed URL, confirm it loads (a 500 here almost always means a
  missing env var — check Step 4).
- If the app has a `vercel.json` cron, confirm it under the project's
  Vercel dashboard → Cron Jobs tab; Vercel Cron only runs on Production
  deployments, not Previews.
- Tell the user it's live, with the URL, and what (if anything) still needs
  manual follow-up outside anything a CLI can do for them — e.g. adding a
  custom domain's DNS records at their registrar, which happens outside
  Vercel/Neon/git entirely and isn't something this skill can do on their
  behalf.

## Later redeploys

Once the checkpoints above have been cleared once for a given app, routine
redeploys are just: commit, push (`git push`), and either let Vercel's
git integration auto-deploy on push, or run `vercel --prod
--token="$VERCEL_TOKEN" --yes` directly. No further confirmation needed for
these unless something about the change itself is risky enough to warrant
it on its own merits (e.g. a destructive migration) — use judgment, same as
any other risky-action call.
