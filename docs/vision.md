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

## Per-project direction

### yggdrasil

Currently a manually curated index. The parked next step is a "second
brain" that doesn't just store state but actively reasons about *why* the
owner made a given decision, gathers supporting evidence, and asks when
it isn't sure — a genuinely different job from being read-and-trusted
context. Not scoped yet: staleness detection (how does it know a stored
belief is outdated?) and memory architecture (what actually lives here
versus in agent-harness's own memory layer?) are both open questions, not
solved ones.

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
- **A dedicated memory layer**: deferred on purpose until manual
  `PROFILE.md`/`PROJECTS.md` curation becomes a real bottleneck rather
  than a mild inconvenience — arguably already true, not yet acted on.
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
has actually been observed with the current file-based memory); a
dedicated human-facing control-plane dashboard (existing logs, a health
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
