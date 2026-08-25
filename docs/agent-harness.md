# agent-harness

A Discord- and web-facing multi-agent orchestrator. You send a message to
an agent — in Discord, or through a real web client — and it drives an
actual Claude Code session against whichever project that agent is
responsible for. Live on native Windows, running under PM2 and Task
Scheduler.

## What "an agent" actually is

An agent is an entry in `agents/registry.json`: a name, a working
directory (`projectDir`), and a persona file (that directory's own
`CLAUDE.md`). There's no separate agent runtime — an agent *is* a Claude
Code session, scoped to a directory, primed with a persona document.

- **Mímir** is the default persona — `projectDir: "."`, the whole
  `agent-harness` repo. Framed explicitly as a secretary/co-owner of the
  system, not a coding assistant scoped to one repo: the first point of
  contact day to day, able to propose recurring routines for itself, and
  expected to have opinions about how the system runs rather than just
  executing isolated requests.
- **Gaia** designs and reviews *other* agents — sandboxed to her own
  `agents/gaia/` directory. She can propose a new agent or an edit to an
  existing one, but can't write `registry.json` or another agent's persona
  file herself; that write is routed to Mímir once a human confirms the
  proposal. Two agents with overlapping write access to the same registry
  would make "who actually approved this" ambiguous — the split keeps
  that answer unambiguous.
- Every other agent is scoped to its own project directory, same pattern.

## Message flow

```
   Discord message              Web client (bus/client)
          │                              │
          └──────────────┬───────────────┘
                          ▼
                 ┌──────────────────┐
                 │   bus/server.js    │   HTTP + WebSocket entry point
                 └─────────┬──────────┘
                           ▼
        ┌───────────────────────────────────┐
        │  session.js / store.js /            │   resolves channel → agent →
        │  channel-state.js                    │   permission mode, history
        └─────────────────┬─────────────────────┘
                          ▼
                 ┌──────────────────┐
                 │   claude-bin.js    │   cmd.exe /c claude <args>
                 └─────────┬──────────┘        (Windows spawn needs this
                           ▼                     exact form — see below)
                 ┌──────────────────┐
                 │ claude-stream.js   │   the "brain" — swappable driver.
                 │  (default brain)   │   wraps a real Claude Code process.
                 └─────────┬──────────┘
                           ▼
              a real Claude Code session runs,
              with that agent's own project directory
              and that agent's own CLAUDE.md persona
                           │
                           ▼
              output streamed back through
              bus → store → client / Discord
```

The bus/store/brain split is deliberate: `store.js` is the persistence
layer (sessions, messages), the bus (`server.js` and friends) is routing
and permission/state resolution, and the "brain" is swappable — the
default is `claude-stream.js`, which spawns and streams a real Claude Code
process, but nothing else in the bus assumes that's the only possible
driver.

## Multi-agent council

When more than one agent's opinion matters for a decision — a review, a
contested call — `council.js` runs a deterministic weighted-vote scoring
pass rather than asking one more LLM to eyeball the disagreement and pick
a winner.

```
       task / review request
                │
     ┌──────────┼──────────┐
     ▼          ▼          ▼
  Agent A    Agent B    Agent C     (independent, blind to each other)
     │          │          │
     └────┬─────┴────┬─────┘
          ▼           ▼
     council.js — deterministic weighted-vote scoring
          │
          ▼
   one reconciled verdict, not majority-vote LLM opinion
```

## Routines — agents proposing their own automation

Any agent can end a reply with a fenced `routine-proposal` block: a name,
a cron schedule, and the exact message to fire when it's due. The client
renders it as a card with a "Create" button — the agent proposes, the
human decides. Once created, a routine fires through the exact same send
path a human message would, which means it inherits that channel's
permission mode, diff tracking, and cost tracking for free — a routine
gets no more capability than a human typing the same message at that
moment would have.

```
  agent reply ends with:
  ```routine-proposal
  {"name": ..., "cron": "0 8 * * *", "prompt": ..., "summary": ...}
  ```
        │
        ▼
  rendered as a card, human clicks "Create"
        │
        ▼
  fires on schedule, addressed to that agent, in that channel —
  same permission mode / tracking as a live human message
```

## Security, honestly stated

Three layers were designed; two are live, the third is a documented,
currently-unclosable gap rather than a hidden one.

```
Layer 1 — Permission mode          default: manual-approval per channel.  LIVE
Layer 2 — Per-agent Bash policy    agents/<name>/settings.json allow/deny  LIVE
Layer 3 — OS-level sandbox         designed (deploy/SANDBOXING.md);        GAP
                                    confirmed not achievable on native
                                    Windows today — Claude Code's own
                                    Windows-sandbox feature exists but is
                                    feature-gated off, not fixable locally.
                                    A real alternative if ever worth the
                                    engineering cost: Windows AppContainer.
```

Layer 1 and 2 were found insufficient once in practice and fixed the same
day the gap was found — the incident and fix live as a comment in the
source, not restated here. The Layer 3 gap is a deliberate accepted
tradeoff (documented, revisit if the platform gate ever lifts), not an
oversight.

## Other real pieces

- **A structured cross-project issue tracker** — outage detection plus a
  real store/API are built; a triage workflow and a dedicated UI tab are
  not.
- **A desktop app** ("Fountain of Mímir") — a Tauri shell around the same
  web client, for a native always-visible presence instead of a browser
  tab.
- **An MCP bridge** — exposes this system's own capabilities as MCP tools,
  so other MCP-aware clients can reach in.

## Windows-specific gotchas worth keeping if you're building something
similar

- `spawn('claude', ...)` does not work directly on Windows — it needs
  `cmd.exe /c claude <args>` with no shell option. Getting this wrong
  produces three different failure modes in sequence (`ENOENT`, then
  `EINVAL`, then a `DEP0190` injection-risk warning) before the right
  invocation is found. Factored into one shared helper
  (`bus/claude-bin.js`) used at every spawn site, rather than re-derived
  per call site.
- Use a real cross-platform UUID source (`crypto.randomUUID()`), not a
  Linux-specific one (`/proc/sys/kernel/random/uuid`) — the latter fails
  silently rather than loudly on Windows, which is worse.
