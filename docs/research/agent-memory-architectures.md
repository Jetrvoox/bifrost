# Agent memory architectures — a survey for ideas worth stealing

Scope: single-agent/session memory *curation* — what gets kept, how it's
organized, what decides ADD vs UPDATE vs DELETE. This is a different layer
from [`shared-state-handoff.md`](shared-state-handoff.md)'s subject
(cross-agent, cross-runtime handoff and provenance) — read that one for
the multi-agent side, this one for the "what does one agent's own memory
actually look like" side. Where the two touch, it's called out below.

Every claim here was checked directly against a primary source (docs,
paper abstract, or repo) at research time, not reconstructed from
training-data familiarity — several of these tools/papers postdate any
model's training cutoff.

## The one directly actionable finding

**Anthropic ships an official memory mechanism for exactly this problem**:
the [`memory_20250818` tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
(Messages API, all Claude 4+ models) plus
[context editing](https://platform.claude.com/docs/en/build-with-claude/context-editing)
and [compaction](https://platform.claude.com/docs/en/build-with-claude/compaction),
designed to work together:

> "compaction keeps the active context small without client-side
> bookkeeping, and memory preserves the information that must survive
> summarization."

That sentence is a clean, literal statement of the exact gap flagged in
`PROJECTS.md`'s memory-layer scoping note — generic compaction (already
inherited here via `--autocompact auto`) handles raw context growth, but
never decides what's *worth* keeping across sessions. The memory tool is
the missing half: Claude itself calls `view`/`create`/`str_replace`/
`insert`/`delete`/`rename` against a `/memories`-scoped file store your
application hosts and executes — client-side, so you control storage,
size caps, expiration, and path-traversal protection.

**Checked directly, resolved**: the Claude Code CLI does *not* expose
`memory_20250818` — `claude --help`'s `--tools` flag documents "the
built-in set" and no `Memory` tool is in it. What the CLI has instead is
its own separate, differently-shaped mechanism: `--bare` mode's own
description names **"auto-memory"** explicitly as a thing it skips,
confirming it's a real, named, built-in feature — CLAUDE.md-adjacent,
index-based rather than semantic. Since `claude-stream.js`'s spawn passes
no `env` override (confirmed earlier), every agent-harness persona already
inherits this exact mechanism — it's the same system that wrote this very
research doc's cross-links. So the real question isn't "can the API tool
be wired in" (it can't, without switching off the CLI-spawn model
entirely, which is a much bigger change than this is worth) — it's
whether each persona is actually using Claude Code's own Auto Memory
well today, which hasn't been separately audited. The API tool's
*interface design* — `view`/`create`/`str_replace`/`insert`/`delete`/
`rename` on a scoped path, the injected "always view memory first, assume
interruption" protocol prompt — is still worth reading as a reference for
shape, just not as something to integrate directly.

Two things worth lifting even without the literal tool:

1. **The injected protocol prompt.** When the tool is present, the API
   auto-adds "ALWAYS VIEW YOUR MEMORY DIRECTORY BEFORE DOING ANYTHING
   ELSE... ASSUME INTERRUPTION: your context window might be reset at any
   moment." That's a one-line standing instruction any persona file here
   could adopt regardless of whether the literal memory tool is wired in.
2. **The documented "multisession software development pattern"**: an
   initializer session sets up a progress log + feature checklist +
   startup-script reference *before* work begins; every later session
   reads those files first; each session updates the log before ending;
   a feature counts as done only after end-to-end verification, not when
   the code is written. This is close to what `PROFILE.md`/`PROJECTS.md`
   already do by convention here, but stated as an explicit, mechanical
   protocol rather than an implicit habit — the "mark done only after
   verification" rule in particular is worth stating explicitly somewhere
   Mímir actually reads it. Anthropic's own case study for this is
   ["Effective harnesses for long-running agents"](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
   — already known here, see local memory `reference_agent_harness_design_patterns`.

## Ideas worth stealing, ranked by how directly they map onto an open gap here

### 1. Mem0's ADD / UPDATE / DELETE / NOOP write-routing

[Mem0](https://github.com/mem0ai/mem0) extracts candidate facts from a
conversation, compares each against its top-k nearest existing memories,
and routes it through an LLM policy that picks one of four operations:
ADD (genuinely new), UPDATE (merges into an existing memory whose
information content increases), DELETE (the new fact contradicts and
supersedes an old one), or NOOP (redundant, do nothing).

**Maps directly onto**: the currently-open "what should the promotion
step actually be" question in `PROJECTS.md`'s memory-review section —
Jetrvoox's own words, "I could say yeah put in memory, but only that part
of the memory, not everything." Mem0's four-way routing is close to a
ready-made answer to that: instead of a proposal being accept/reject as a
whole file, each proposal could be diffed against the existing memory it's
touching and classified the same way — a real design pattern to borrow
for `bus/routes/memory-staging.js`'s eventual promotion logic, not just
inspiration.

Worth noting as a caution too: Mem0's own v3 (April 2026) walked this
back toward simpler single-pass ADD-only extraction with cross-memory
entity linking — a real signal that the four-way LLM-routed consolidation
was more machinery than it was worth at their scale. Don't over-build the
first version of this here either.

### 2. SAGE — a cheap novelty gate in front of the expensive LLM call

[SAGE](https://arxiv.org/abs/2605.30711) (Duke, 2026) frames memory writes
as a novelty-detection problem: score each candidate fact with a cheap
statistical density estimate over existing memory embeddings first, and
only send genuinely ambiguous cases to an LLM for the ADD/merge decision.
Clearly-novel facts get auto-ADD, clearly-redundant ones get auto-NOOP,
with no LLM call at all. Reported effect as a drop-in gate in front of
Mem0/A-Mem: 3.4× lower add-phase API cost, 2.5× lower latency, ~16-18%
of LLM calls skipped entirely, small quality cost.

**Maps onto**: this is the *general form* of a pattern already built and
proven here — `reference_bot_chat_loop_prevention_pattern`'s "turn budget
as backstop + self-assessed novelty as the real gate" is functionally the
same idea (cheap/structural check first, model judgment only when
ambiguous), just applied to a different problem. Worth applying that same
shape to the memory-staging pipeline: a cheap embedding-similarity check
against recently-staged/promoted memories before a `memory-proposal`
block even reaches the review channel, rather than every parsed proposal
generating a card a human has to look at.

### 3. Zep/Graphiti — invalidate, don't delete, on a bi-temporal edge

[Zep](https://arxiv.org/abs/2501.13956) represents memory as a temporal
knowledge graph: episodic nodes (raw messages) → extracted entities/facts
as graph edges, each edge carrying *two* timestamps (when the fact became
true in the world, when the system learned it) rather than one. When a
new fact contradicts an old one, the old edge's validity window is closed
(`valid_to` set) rather than deleted — the fact stays queryable ("what did
we believe in January"), and every answer traces back to the episode that
produced it.

**Maps onto**: this is close to a concrete, already-shipped implementation
of the "tape" idea `shared-state-handoff.md` already identified as the
genuine, unaddressed gap here — "a queryable append-only log of which
agent ran with which promoted item in context." Zep is real prior art for
that exact shape, not just an adjacent idea — worth reading Graphiti's
actual schema as a reference design if that tape ever gets built, rather
than designing the supersession/validity-window mechanics from scratch.

### 4. A-Mem — new memories retroactively re-link old ones

[A-Mem](https://arxiv.org/abs/2502.12110) (NeurIPS 2025) organizes memory
as a Zettelkasten-style network: each new memory gets structured
attributes (context, keywords, tags), the system searches existing
memories for genuine connections, and — the actually novel part — **a new
memory can trigger updates to the attributes/links of existing older
memories**, not just link forward to them. Reported 85-93% lower memory-
operation token usage and up to 6× better multi-hop reasoning vs. static-
link baselines.

**Maps onto**: this memory system already uses `[[name]]` wikilinks
between memory files, explicitly encouraged to "link liberally." But that
linking is one-directional by convention — a new memory references an old
one; nothing ever goes back and updates the old memory to reflect that a
newer, related one now exists. A-Mem's idea is the missing half: when
writing or promoting a memory, it's worth explicitly checking whether any
*existing* memory should gain a link to the new one, not just leaving the
graph forward-only. Small, concrete, and directly applicable without
adopting anything else about A-Mem's architecture.

### 5. Generative Agents — retrieval scoring + periodic reflection

The original [Park et al. Stanford paper](https://dl.acm.org/doi/fullHtml/10.1145/3586183.3606763)
retrieves memories by a weighted score — `recency (exponential decay) +
importance (self-assessed) + relevance (embedding similarity)` — and
separately runs a periodic **reflection** pass: cluster related raw
observations, synthesize a higher-level insight, and feed that insight
back into the memory stream alongside the raw observations it came from.

**Maps onto**: this is the same instinct as Letta's sleep-time compute
(already covered in `shared-state-handoff.md`) but with a concrete,
implementable scoring formula and an explicit two-tier structure (raw
observations vs. synthesized reflections, both retrievable, both dated).
If a "digest"/consolidation routine ever gets built here — turning a
week of staged proposals + routine outputs into a few durable PROJECTS.md
entries instead of leaving Jetrvoox to do that read manually — this
scoring function is a reasonable starting point rather than inventing
one from scratch.

## What this doesn't change

None of the above argues for a vector-DB/RAG memory layer as a whole —
ultracommands' own explicit, considered decision ("no retrieval-quality
problem has actually been observed with the current file-based memory")
still stands; several of these ideas (Mem0's routing, SAGE's gate, A-Mem's
backlinking) are about *write-time curation logic*, not about swapping the
storage substrate, and apply just as well to plain markdown files as to an
embedding store. The actionable short list, smallest first: (1) A-Mem's
backlink-old-memories habit — no infrastructure change, just a practice;
(2) SAGE-style cheap novelty check before a memory-proposal card reaches
the review channel; (3) Mem0-style ADD/UPDATE/DELETE/NOOP as the real
answer to the still-open partial-promotion question; (4) auditing whether
each agent-harness persona is actually using Claude Code's own built-in
Auto Memory well — confirmed real and already inherited by every spawned
session, not something to build or bolt on, just something to check.
