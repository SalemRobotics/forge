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
  following both of the above. Local-only; touches no cloud resources.
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

From inside Claude Code, in any project:

```
/plugin install 6324-app-stack@local
```

or, if you'd rather not install it as a plugin, you can just copy
`skills/` and `agents/` straight into `~/.claude/skills/` and
`~/.claude/agents/` — they're plain markdown, no build step.

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
