# Vision

What "done" looks like for this ecosystem, and — just as importantly —
what's deliberately not being built yet, and why. Every open item below
is either explicitly scoped for a future session or explicitly gated on a
trigger condition that hasn't fired. Nothing here is speculative
roadmap-as-marketing; it's the real backlog.

## The target shape

```
                    ┌───────────────────────────────┐
                    │            yggdrasil             │
                    │   index today  →  active "second │
                    │   brain" tomorrow: reasons about   │
                    │   *why* the owner acts, gathers      │
                    │   evidence, asks when unsure          │
                    └────────────────┬──────────────────┘
                                     │
                  ┌────────────────────┴────────────────────┐
                  ▼                                           ▼
      ┌─────────────────────────┐                 ┌─────────────────────────┐
      │      agent-harness         │◄───shared──────►│       ultracommands        │
      │  brain-swap / cross-        │    memory        │   capability registry       │
      │  project router (which      │    layer          │   (trigger: 20+ tools,       │
      │  brain handles which         │  (not yet built,  │    real routing conflicts)   │
      │  message) — not yet built    │   manual index    │                                │
      │                               │   is the current  │                                │
      │                               │   bottleneck)      │                                │
      └─────────────────────────┘                 └─────────────────────────┘
                  │                                           │
                  └────────────────────┬────────────────────┘
                                       ▼
                     autoresearch loop: baseline → modify one
                     aspect of the environment → run the
                     benchmark suite → keep or revert
                     (deliberately LAST — needs a benchmark
                      mature enough to trust before it's safe
                      to run unattended)
```

## Closed since last update

One item below stopped being open. Noted here rather than silently
dropped from the list, since the gap it closed was named explicitly
enough elsewhere in this repo that leaving it in a "not yet built" list
would read as still-true when it isn't:

- **agent-harness's dedicated memory layer** (2026-09-04) — the harder,
  unsolved half of memory this doc used to describe (deciding what's
  worth keeping across a reset, not just compacting raw context) is real
  now: a tape → digest → resume pipeline. Full detail in
  [`docs/agent-harness.md`](agent-harness.md#memory-across-resets--tape-digest-resume).
  What's still open, correctly scoped as v2 rather than a gap in the
  mechanism: a human-facing digest-browsing UI. The comparative survey
  that motivated this item stays useful background even with the
  immediate gap closed:
  [`docs/research/agent-memory-architectures.md`](research/agent-memory-architectures.md).

## Per-project direction

### yggdrasil

Currently a manually curated index — see
[`docs/yggdrasil.md`](yggdrasil.md) for how that index itself was
restructured in September once a single growing file stopped being cheap
to read. The parked next step is a "second brain" that doesn't just store
state but actively reasons about *why* the owner made a given decision,
gathers supporting evidence, and asks when it isn't sure — a genuinely
different job from being read-and-trusted context. Still explicitly
parked pending a stated trigger (not started early), but real design work
landed while it waited: a raw/synthesized-wiki split borrowed from outside
research, evaluated and confirmed correct rather than just plausible — the
value isn't stylistic, it's that a bad AI-authored edit stays recoverable
by re-deriving from the untouched raw layer instead of permanently
corrupting the only copy of a claim. Staleness detection (how does it know
a stored belief is outdated?) and the boundary against agent-harness's own
memory layer above are both still open questions, not solved ones.

### agent-harness

- **Issue tracker, phases 3-4**: a triage workflow and a dedicated UI tab.
  Outage detection and the underlying store/API (phases 1-2) are live.
- **Council v2**: sequential/"aware" fan-out instead of fully blind
  parallel voting, plus a way to drill into a real dissent in Discord
  instead of only seeing the reconciled verdict.
- **A brain-swap / cross-project router**: right now, which brain handles
  a message is fixed per agent. The open question is whether a dedicated
  router agent should decide that dynamically instead — not yet decided
  who or what owns that decision.
- **A per-user model** (new direction, started 2026-09-05; not the same
  thing as the memory layer resolved above): that layer governs one
  channel's continuity across a reset; this is about what Mímir actually
  learns about one *specific person* over time, versus applying the same
  learned behavior to everyone talking to it. Early and deliberately
  paced — see [`docs/agent-harness.md`](agent-harness.md#a-per-user-model-just-starting)
  for exactly how little is real yet and why it's staying that way for
  now (a stated anti-pattern in the underlying design: don't add a second
  memory mechanism before the first feedback loop — model → context →
  signal → weekly review → model — is actually closed and has proven
  itself).
- **A real git client inside the client workspace**: branch switch, commit
  history browsing, staging/committing, and repo browsing at a given
  commit, not just the existing per-turn diff view. Scoped, not built —
  and flagged on purpose as a materially bigger trust surface than
  anything else on this list, since it's a second place able to mutate
  real repo state rather than only observe it. Needs its own decision on
  confirmation steps before any of it ships.
- **A persona upgrade for the default assistant**: give it a voice and
  substance shaped by this system's own accumulated work, instead of a
  generic assistant tone bolted onto a system-specific job description.
- **Dynamic model routing by task complexity**: every agent currently runs
  at one fixed model/effort setting, uniform across all agents and turns.
  The idea is picking model/effort per turn based on how complex the
  actual task looks, rather than one static setting for everything.
- **Fanout worker dispatch**: pass a set of genuinely independent
  sub-tasks instead of one, run one worker per entry in parallel, resolve
  as a group, with each worker's live cost/token numbers surfaced as its
  own card. Real design work done, not yet built — the open question was
  mechanism (a directive the bus itself parses and clamps, not a tool a
  session could hand looser permissions to on its own) and process shape
  (each worker as its own standalone process, the only way to get honest
  live-per-worker cost data rather than something bundled and delayed).
  Claude Code's own delegate-vs-fork split and its harness-enforced
  (not prompt-enforced) tool-pool filtering are directly relevant prior
  art here — see
  [`docs/research/agent-infra-ecosystem-survey.md`](research/agent-infra-ecosystem-survey.md#5-fork-full-context-vs-delegate-isolated-summary-only-are-different-primitives).

### ultracommands

Three remaining commands from the original backlog, plus four larger
ideas evaluated and explicitly trigger-gated rather than built
speculatively:

1. **Capability/tool registry** — route by capability
   (`vision.inspect_video`) instead of by which integration happens to
   implement it. Trigger: 20+ tools with real routing conflicts. Not
   worth it yet at current scale.
2. **Karpathy-style autoresearch loop** — baseline, modify one aspect of
   the environment, run the benchmark suite, keep or revert. Trigger: the
   benchmark suite has enough real logged runs to detect a regression
   confidently. Deliberately the *last* thing to build here — it needs a
   benchmark worth protecting before it's meaningful.
3. **Cross-provider model routing** — quality × cost × latency across
   providers, not just effort tier within one model family. Trigger: team
   scale. Per-role effort tuning (already shipped) is judged the
   correctly-sized version of this idea for a solo-maintained framework.
4. **Perception layer as a formal capability abstraction** — vision/
   video/audio routed through a registry with provider fallback. Trigger:
   actually having competing implementations of the same modality, which
   doesn't exist yet — one working pipeline already covers the real need.

Explicitly declined, not just deferred: a standing capability audit baked
into permanent command logic (a one-time document does this job fine —
recurring logic would be solving a problem that isn't recurring); a
knowledge-graph or vector-RAG memory layer (no retrieval-quality problem
has actually been observed with the current file-based memory — and if
that trigger ever fires, the reference shape to reach for is deferred,
budget-gated graph construction rather than eager extraction, see
[`docs/research/agent-infra-ecosystem-survey.md`](research/agent-infra-ecosystem-survey.md#1-defer-expensive-structure-dependent-computation-to-query-time-budget-gated));
a dedicated human-facing control-plane dashboard (existing logs, a health
endpoint, and agent-harness's own health panel already cover this at
current scale).

## The throughline

Every deferred item above shares the same shape: a real idea, with a
stated concrete trigger, left unbuilt until that trigger actually fires.
That's a deliberate stance, not a lack of ambition — building ahead of a
real need is exactly how a solo-maintained system accumulates unused
complexity it then has to carry forever. The bet is that staying small
until forced to grow produces a system that's actually understood by the
one person maintaining it, at every point along the way.
