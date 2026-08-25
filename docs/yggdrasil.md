# yggdrasil

The entry point. Every Claude Code session on this system reads this repo
before doing anything else — not just sessions already working inside a
related project.

## What it is

`yggdrasil` holds no secrets and no generated code. It's context, not a
codebase: two files (`PROFILE.md`, `PROJECTS.md`) that make a fresh
session immediately useful instead of starting from zero.

```
        new Claude Code session starts, anywhere on the machine
                              │
                              ▼
                  ┌────────────────────────┐
                  │      yggdrasil/CLAUDE.md │   "read me first"
                  └────────────┬────────────┘
                               │
                 ┌─────────────┴─────────────┐
                 ▼                             ▼
      ┌────────────────────┐        ┌────────────────────┐
      │     PROFILE.md       │        │     PROJECTS.md      │
      │  who the owner is,   │        │  index of active     │
      │  how they like to    │        │  projects, repos,    │
      │  work, hard          │        │  and current status  │
      │  constraints          │        │                       │
      └────────────────────┘        └────────────────────┘
                 │                             │
                 └─────────────┬─────────────┘
                               ▼
                session now grounded — knows who it's
                working for and what state everything
                is in, without asking
```

## The one exception to "no code"

`claude-code/` is a snapshot of one machine's actual Claude Code
environment — plugins, hooks, scripts, skills, settings — plus an
installer, so a new machine can adopt the same setup instead of being
configured by hand. Everything else in the repo is prose.

## Why a separate repo instead of folding this into each project

A project's own `CLAUDE.md` should describe *that project*. Cross-project
facts — who the owner is, what else exists, what changed recently
elsewhere — don't belong duplicated into every repo, and duplicating them
guarantees drift the moment one copy gets updated and the others don't.
One index, read first, avoids that.

## Maintenance discipline

When a project's status materially changes — a stage ships, a decision
gets made, a project wraps up — the session that did the work updates
`PROJECTS.md` before moving on, so the next session (possibly weeks later,
possibly a different one entirely) doesn't inherit stale state. Getting
this wrong has real cost: a stale status entry once caused a separate
session to independently rebuild something that already existed, discovered
only when a routine `git fetch` surfaced the divergence right before a
push. The fix wasn't a process document — it was tightening the actual
habit of updating the index the same day work lands.

## Some project-internal decision logs this doesn't replace

Not every decision belongs in the cross-project index. Two of the projects
below keep their own local log instead, closer to the code the decision
actually affects:

- **ultracommands** keeps a dated, incident-tied "lessons learned" section
  directly in its own `CLAUDE.md` — framework-internal design calls, not
  duplicated here.
- **agent-harness** keeps no separate log file at all — real design
  decisions live as dated, incident-citing comments directly in the
  source they affect.

Both are deliberate: a decision log next to the code it explains stays
accurate for free, the way a copy filed somewhere else does not.
