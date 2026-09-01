# Agent-infra ecosystem survey — patterns worth stealing from 20 external projects

Scope: not memory curation specifically (see
[`agent-memory-architectures.md`](agent-memory-architectures.md) for that) —
this is the broader landscape: task orchestration, context/token economics,
tool/capability brokering, RAG, browser agents, observability, and how
Claude Code itself is built. Run as seven independent research passes, each
briefed to read actual source (GitHub file trees, real code) rather than
recall from training or trust a README — confidence is marked per claim
where it matters (`verified-source` / `docs-level` / `unverified`) in the
private working version of this survey; only claims worth real confidence
are repeated here.

## The one law that showed up twice, independently

Two unrelated Rust codebases — **Headroom** (a context-compression proxy)
and **RTK** (a shell-output filter) — independently arrived at the identical
invariant: if a compressed/filtered version of something is larger than the
raw version, discard the compression and return raw. Call it `never_worse()`.
Treat it as close to a physical law for this problem class, not a stylistic
choice either project happened to make. Worth stating explicitly anywhere
this ecosystem ever builds a compression or filtering layer, rather than
rediscovering it the hard way.

## Ideas worth stealing, mapped onto real open questions here

### 1. Defer expensive structure-dependent computation to query time, budget-gated

[GraphRAG](https://github.com/microsoft/graphrag) builds an entity-
relationship graph eagerly at index time — expensive (widely cited at
3-5x vector-RAG indexing cost, and the mechanism is now traceable: every
chunk pays for a base LLM extraction pass plus a self-judged "did you miss
anything, Y/N" gleaning loop that can run several rounds). Microsoft's own
follow-up, **LazyGraphRAG**, inverts this: index-time cost drops to ~0.1% of
full GraphRAG (cheap NLP only), and all LLM-driven graph reasoning moves to
query time, gated by one tunable "relevance test budget" knob. Reported to
match full-GraphRAG quality at roughly 4% of its query cost.

**Maps onto**: `ultracommands`' explicit, already-considered decision
against a knowledge-graph or vector-RAG memory layer ("no retrieval-quality
problem has actually been observed with the current file-based memory," see
[`vision.md`](../vision.md)) still stands — this doesn't argue against that.
But it's worth recording the nuance for if that trigger ever does fire:
LazyGraphRAG's deferred-compute shape is the one to reach for, not eager
GraphRAG. The general pattern — defer expensive structure-dependent
computation to query time, gated by one tunable cost/quality parameter,
rather than paying for it unconditionally on every write — is also a
reasonable lens for agent-harness's own still-open curated-memory-layer
question in [`agent-memory-architectures.md`](agent-memory-architectures.md).

### 2. A gate is a dependency edge, not something to poll

[Beads](https://github.com/gastownhall/beads) (a SQL-backed, Dolt-versioned
task DAG for multi-agent work) turns "wait for CI / a human / a timer" into
an ordinary, closeable dependency edge — an agent calls `bd ready` and
either it's blocked or it isn't, no polling, no separate wait-primitive.
Its dependency taxonomy is explicit and typed (`blocks`, `conditional-blocks`
for error-branch handling, `waits-for` for dynamic fan-in over children not
known at authoring time, non-blocking `related`) rather than the usual
one-edge-type collapse.

**Maps onto**: agent-harness's issue tracker has outage detection and a real
store/API live, but no triage workflow yet (see
[`agent-harness.md`](../agent-harness.md)). If that tracker ever grows
dependencies between issues, this typed-edge model — especially gate-as-
dependency-edge — is a more agent-native shape than a flat status field.

### 3. Least-privilege your own internal tooling, not just the external boundary

[Composio](https://github.com/ComposioHQ/composio) (a hosted OAuth/API
broker for ~1000 external tools) is built on a genuinely good idea —
credentials never enter the LLM's context window; the model names an
action, a backend resolves and injects the credential server-side. In May
2026 it suffered a real breach anyway: a compromised employee's *personal*
Gmail OAuth token reached an internal agent-ops monitoring tool with broad
standing privilege, which had an ungoverned path into the same credential
cache the well-designed per-tenant model was supposed to protect. The
external-facing isolation was never actually breached directly — the
failure was one layer up, in Composio's own internal tooling.

**Maps onto**: agent-harness's own security section is honest about a real,
disclosed gap (Layer 3, OS-level sandboxing, currently unachievable on
native Windows — see [`agent-harness.md`](../agent-harness.md)). This is a
different, complementary lesson worth logging alongside it: whatever
internal automation touches Discord bot tokens, GitHub tokens, or other
brokered credentials deserves the same zero-standing-privilege,
per-action-authorization discipline this system already applies at the
external boundary (Layer 1/2, identity roles, `minRole` gating) — not
assumed by extension just because the external model is sound.

### 4. Reasoning-blind safety classifiers close the injection channel structurally

Claude Code's own Auto Mode classifier (Anthropic's published design) is
deliberately blind to reasoning and tool outputs — it judges only user
messages and executable tool-call payloads, with Claude's own explanatory
text and *all* tool-output content stripped before the classifier ever sees
it. This closes the exact channel indirect prompt injection travels through,
instead of trying to teach a classifier to distinguish legitimate from
injected content within that channel. It's two-stage for cost: a cheap,
deliberately over-blocking filter (~8.5% false-positive, tuned for recall)
escalates only flagged actions to a full reasoning pass.

Anthropic's own stated priority, worth repeating verbatim because it's the
sharpest single sentence in the whole survey: *design for containment at the
environment layer first, then steer behavior at the model layer.* Two of
their own disclosed incidents were both cases of data leaving through an
*allowlisted* destination via an unauthorized request — "the model layer
couldn't help." And a named, permanent limit: none of this helps when the
user themselves is the one who typed the malicious instruction (an internal
red-team phishing case succeeded 24 times out of 25).

**Maps onto**: no forced mapping here — agent-harness doesn't currently run
a safety classifier of its own — but it's a directly relevant reference
design the moment prompt-injection-from-external-content becomes a live
concern (e.g. if `bot-to-bot conversation` or the MCP bridge ever ingests
untrusted third-party content). Filed as a pattern to reach for, not a gap
to fix now.

### 5. Fork (full context) vs. delegate (isolated, summary-only) are different primitives

Claude Code's own subagent model treats these as genuinely distinct: a
delegated subagent gets an isolated context (its own system prompt, the
delegation message, no parent conversation history) and returns a summary;
a **fork** inherits the entire parent context verbatim — closer to process
`fork()` semantics than a worker. Tool-pool access is filtered in two
mechanical stages (a universal strip, then a background-only strip),
enforced by the harness regardless of what the subagent's own prompt says.

**Maps onto**: this is close to a ready-made vocabulary for the still-open
**fanout worker dispatch** idea in [`vision.md`](../vision.md) — passing a
set of genuinely independent sub-tasks to parallel workers, each with its
own live cost/token card. The open question there was mechanism (a directive
the bus itself parses and clamps) and process shape (each worker as its own
standalone process for honest live-per-worker cost data). Claude Code's
delegate/fork split and its harness-enforced (not prompt-enforced) tool
filtering are both directly relevant prior art if that gets built.

### 6. Progressive disclosure as the general shape for "index vs. body"

The [Agent Skills spec](https://agentskills.io/specification) (now an open,
cross-vendor standard, not Anthropic-proprietary) loads a skill in three
tiers: metadata always resident (~100 tokens), full instructions only once
the model's own reasoning judges relevance, resources (scripts/references)
at zero cost until actually opened — and a script's *source* never enters
context, only its stdout. A concrete, non-obvious failure mode: reference
chains nested more than one hop from the entry file get silently truncated
(previewed via a `head`-style partial read, not fully read).

**Maps onto**: this is the same "docs as a map, not an encyclopedia"
philosophy already stated in this repo's own [README](../../README.md), and
directly relevant to `yggdrasil`'s own PROFILE/PROJECTS structure and any
future `ultracommands` skill work. The one-hop-reference rule in particular
is worth checking against explicitly — it's a concrete, testable failure
mode, not just a style preference.

## What this doesn't change

None of this argues for adopting any of these projects wholesale — most of
what's stealable here is a pattern or an invariant, not a dependency to add.
`ultracommands`' file-based-memory decision and agent-harness's Layer 1/2
security model both still stand as designed. The actionable short list,
smallest first: (1) `never_worse()` as a standing rule for any future
compression/filtering work; (2) logging Composio's internal-tooling lesson
against agent-harness's own credential-handling surface, not just its
external-facing security layers; (3) reading Claude Code's delegate/fork
split before designing the fanout-worker dispatch mechanism, rather than
inventing the vocabulary from scratch; (4) treating LazyGraphRAG, not eager
GraphRAG, as the reference shape if the file-based-memory decision is ever
revisited.
