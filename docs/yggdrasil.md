# yggdrasil

The entry point. Every Claude Code session on this system reads this repo
before doing anything else — not just sessions already working inside a
related project.

## What it is

`yggdrasil` holds no secrets and no generated code. It's context, not a
codebase: four files today that make a fresh session immediately useful
instead of starting from zero. Started as `PROFILE.md` + `PROJECTS.md`
(2026-08-08), grew `PROCESS.md` (2026-08-13) and, later, a `TODO.md`
(2026-08-22). The most recent change, 2026-09-02, replaced `TODO.md` with
`ROADMAP.md` and split `PROJECTS.md`'s single growing file into a slim
index plus one file per project — see "Why it's four files now" below for
the real incidents behind both.

```
        new Claude Code session starts, anywhere on the machine
                              │
                              ▼
                  ┌────────────────────────┐
                  │      yggdrasil/CLAUDE.md │   "read me first"
                  └────────────┬────────────┘
                               │
       ┌───────────────┬───────────────┬───────────────┐
       ▼               ▼               ▼               ▼
 ┌───────────┐   ┌───────────┐   ┌───────────┐   ┌───────────┐
 │ PROFILE.md  │   │ ROADMAP.md  │   │ PROJECTS.md │   │ PROCESS.md  │
 │ who + how +  │   │ every open   │   │ slim index;  │   │ how to work  │
 │ hard          │   │ item, one     │   │ full detail   │   │ across        │
 │ constraints    │   │ source of      │   │ per project    │   │ projects       │
 └───────────┘   └───────────┘   └───────────┘   └───────────┘
       │               │               │               │
       └───────────────┴───────────────┴───────────────┘
                               ▼
                session now grounded — knows who it's working
                for, what's open, what state everything is in,
                and how this owner expects work to be done —
                without asking
```

`PROJECTS.md`'s "full detail per project" lives in `projects/<name>.md`,
split out once the single file passed 1600 lines (see "Why it's four files
now" below) — read one directly once it's known which project matters, or
pull the relevant slice via the `memory-search` MCP tool's `search_memory`
(`source: "yggdrasil"`) rather than reading every project file by default.

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
`PROJECTS.md`'s index entry and the relevant `projects/<name>.md` before
moving on, so the next session (possibly weeks later, possibly a different
one entirely) doesn't inherit stale state. Getting this wrong has real
cost: a stale status entry once caused a separate session to independently
rebuild something that already existed, discovered only when a routine
`git fetch` surfaced the divergence right before a push. The fix wasn't a
process document — it was tightening the actual habit of updating the
index the same day work lands.

## Why it's four files now

Both splits came from a real, named incident, not a tidiness pass:

- **`PROJECTS.md` → slim index + `projects/<name>.md`, 2026-09-02.** The
  single file passed 1600 lines and even a "what's the state of
  everything" fallback read had stopped being cheap. Splitting it doesn't
  change what's tracked, only where the *narrative* for one project lives
  versus the cross-project snapshot.
- **`TODO.md` → `ROADMAP.md`, same day.** Open ideas, deferred decisions,
  and parked items had been scattered across `PROJECTS.md`'s own
  narrative prose instead of living in one place — a real Wake-on-LAN idea
  sat buried in one project's status note for weeks before it was
  recovered by mining an old session transcript, not by anyone reading
  `PROJECTS.md` and noticing it. `ROADMAP.md` is now the single place
  every open item lives, each tagged `open` / `blocked` / `parked`; a
  retired item moves to `ROADMAP-DONE.md` with a completion date rather
  than being deleted outright, so the record survives without needing
  git-log archaeology to reconstruct.

`PROCESS.md` (standing practices — scope-first, memory single-source-of-
truth, worktree isolation in shared repos, the roadmap-sync rule that
keeps the above two in sync) predates this split but is worth naming here
too: it's an unconditional full read every session specifically because
skipping it once already caused a real mistake.

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
