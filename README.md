# bifrost

A public snapshot of a personal, evolving agent ecosystem built on top of
[Claude Code](https://claude.com/claude-code). This repo holds no code and
no secrets — it's documentation, kept manually up to date, describing how
three private repos fit together and where they're headed.

Named `bifrost` because the actual working repo is called `yggdrasil` (the
world tree — the private index everything else grows from). This is the
bridge out: a curated view of the tree, safe to hand to someone else's AI
without handing over the tree itself.

## The three pillars

```
                         ┌─────────────────────────┐
                         │         human owner       │
                         │  direction + approval      │
                         │  gates on anything          │
                         │  external-facing             │
                         └────────────┬────────────┘
                                      │
                                      ▼
                         ┌─────────────────────────┐
                         │        yggdrasil          │
                         │  PROFILE + PROJECTS index │
                         │  "read me first"           │
                         └────────────┬────────────┘
                                      │  every session starts here
                    ┌─────────────────┴─────────────────┐
                    ▼                                     ▼
      ┌───────────────────────────┐         ┌───────────────────────────┐
      │       agent-harness         │         │        ultracommands       │
      │  Discord/web multi-agent    │         │  skill + Workflow command   │
      │  orchestrator, always-on    │         │  framework, project-local   │
      └──────────────┬─────────────┘         └──────────────┬─────────────┘
                      │  spawns                                │  dispatches
                      ▼                                        ▼
              ┌────────────────┐                       ┌────────────────┐
              │  Claude Code    │                       │  Claude Code    │
              │   sessions      │      same underlying   │   sessions      │
              └────────────────┘      CLI, different     └────────────────┘
                                       entry points
```

- **[yggdrasil](docs/yggdrasil.md)** — the index. Not a codebase; the file
  any session reads first so it doesn't start from zero.
- **[agent-harness](docs/agent-harness.md)** — a Discord/web-facing
  multi-agent orchestrator. Text an agent, it drives a real Claude Code
  session against whichever project that agent owns.
- **[ultracommands](docs/ultracommands.md)** — a project-local command
  framework (`/ultracode`, `/ultraplan`, `/ultra-review-local`, ...) built
  entirely from officially supported Claude Code mechanisms, reproducing
  the *observable behavior* of a deeper built-in review pipeline without
  touching any proprietary implementation detail.

See **[vision.md](docs/vision.md)** for where this is headed, and what's
deliberately not built yet.

## Design philosophy, applied consistently across all three

- **Compose, don't duplicate.** A new command reuses another command's
  existing procedure instead of re-implementing it inline. A new memory
  need routes into an existing store instead of spawning a new one.
- **Docs as a map, not an encyclopedia.** Point at where a decision lives
  (often directly in the code, as a dated comment) instead of copying it
  into a second place that will drift.
- **Human approval is structural, not a suggestion.** Anything
  external-facing — a message sent, a call placed, a deploy pushed —
  needs an explicit human step, logged somewhere durable. This is a fixed
  property of the system, not a setting.
- **Build on trigger, not on speculation.** Several real future-facing
  ideas (see [vision.md](docs/vision.md)) are deliberately left unbuilt
  until a concrete trigger condition is actually met — e.g. a capability
  registry isn't worth building until there are enough tools to actually
  have routing conflicts.

## What this repo is not

- Not a mirror of the private repos — no source code, no commit history,
  no credentials, no client/business data.
- Not automatically synced — updated by hand after material changes
  elsewhere in the ecosystem.
