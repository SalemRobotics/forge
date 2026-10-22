# 6324 App Stack

A Claude Code plugin that packages up how team 6324 builds and deploys
internal tools, so every new tool (attendance, inventory, scouting,
whatever's next) looks and behaves the same way, and ships the same way,
without re-deriving the conventions from scratch each time.

This started from the `frc-attendance` app — that repo is still the
canonical, living reference implementation. Everything in here is either a
copy of its documented standards or an automation of the steps that were
done by hand to build and ship it.

## What's in here

- **`references/ui-standards.md`** — the visual/component design system
  (colors, typography, component catalog, accessibility rules). Written as
  "6324 Generative Tool UI Standards."
- **`references/stack-conventions.md`** — the backend/infra conventions:
  repo layout, schema/migration rules, the auth pattern, the server-action
  pattern, the testing pattern, and the deploy checklist.
- **`skills/scaffold-6324-app/`** — scaffolds a brand-new app from scratch
  following both of the above, for any domain (not just attendance-shaped
  tools). Local-only; touches no cloud resources.
- **`templates/`** — the handful of files that are genuinely identical
  across every 6324-stack app (the DB client, `drizzle.config.ts`, the
  server-action helpers) and so get copied verbatim by the scaffold skill
  instead of rewritten each time.
- **`skills/deploy-6324-app/`** — deploys a 6324-stack app to Vercel + Neon
  using the Vercel CLI, Neon CLI (`neonctl`), and `git`, installing those
  CLIs if they're missing. Pauses for your explicit confirmation before the
  three moments that are hard to undo (first push to a shared remote, first
  production deploy, creating a billed cloud resource) — everything else
  runs straight through without asking each time.
- **`agents/stack-builder.md`** — a subagent for implementing one
  well-scoped feature inside an existing 6324-stack app, following the same
  conventions. Deliberately does *not* handle deployment — that stays in
  the main conversation so the confirmation checkpoints above actually work.

## Installing

This repo is both a plugin (`.claude-plugin/plugin.json`) and its own
local marketplace (`.claude-plugin/marketplace.json`), so it's
self-contained — no separate marketplace repo needed, and no need to be
published anywhere first. That matters because the intended distribution
is: **clone this repo once, then add it as a git submodule inside each
project that wants it** (`git submodule add <this-repo-url> .6324-app-stack`
or similar), and point Claude Code at that local submodule path.

Register the marketplace, pointing at wherever this repo (or the submodule)
lives on disk — relative paths resolve from your current shell/session
working directory, so either `cd` there first or use an absolute path:

```
/plugin marketplace add ./path/to/6324-app-stack
```

Then install it:

```
/plugin install 6324-app-stack@6324-tools
```

(`6324-tools` is this repo's marketplace name, from `marketplace.json`;
`6324-app-stack` is the plugin name, from `plugin.json` — they're
deliberately different strings so the `name@marketplace` syntax reads
clearly.)

If you'd rather skip the plugin system entirely, you can also just copy
`skills/` and `agents/` straight into `~/.claude/skills/` and
`~/.claude/agents/` — they're plain markdown, no build step, and that
works whether or not this repo is a submodule anywhere.

## Credentials this expects you to set up yourself

Neither skill will ever try to obtain, generate, or interactively log in
for these on your behalf — if they're missing, you'll be told exactly where
to get them and the deploy skill will stop and wait.

| Variable | Where to get it |
|---|---|
| `VERCEL_TOKEN` | Vercel dashboard → Account Settings → Tokens → Create Token |
| `NEON_API_KEY` | Neon console → Account Settings → API Keys → Create API Key |
| GitHub auth (for `gh`) | `gh auth login` (one-time, interactive) |

## Updating this plugin

When the attendance app (or any other 6324 tool) establishes a new pattern
worth generalizing, update the reference docs here to match, bump
`.claude-plugin/plugin.json`'s version, and commit. This repo should stay a
living document, not a one-time snapshot.
