# Shared state & cross-agent handoff — research artifact

Produced 2026-08-25 via `ultracommands`' `/ultraresearch` (4 angles, parallel
researcher agents, WebSearch/WebFetch, high-effort synthesis reconciliation).
Full run: 65 tool calls, 310k tokens, all 4 angles + synthesis completed
clean (no `done_unsynthesized` fallback needed).

Formatted as artifact candidates, not prose — each item carries a source and
a confidence rating so it can be evaluated on its own merits at promotion
time, the same discipline this doc's own topic is about. Nothing here has
been promoted anywhere; this is the raw material for that decision, not the
decision itself. Only `strong` and well-corroborated `moderate` items are
included — the full run (with `weak` items and the complete finding list)
is available on request if useful.

## Headline item

**`adr-kit`** (github.com/rvdbreemen/adr-kit, MIT, ~483 commits, v0.52.0) is
a real, shipping Claude Code plugin that implements close to exactly the
artifacts → pins → tape shape: append-only Markdown ADRs (each with a YAML
Status History block — date/changed_by/reason/changed_via — appended, never
overwritten) are the ground-truth tape; a compact `ADR-INDEX.md` is what
actually gets injected at session cold-boot; a richer `ADR-INDEX.json`
(supersession graph, drift-detection fingerprints) is queried on-demand
rather than loaded wholesale. Promotion is human-only: agents may draft
`Proposed`-status records or print advisory nudges, but only an explicit
same-session human "yes" moves anything to `Accepted`. Docs state directly:
*"source material is evidence; it is never acceptance authority."*
**Confidence: moderate** (single maintainer's project, not enterprise-scale
verified, but the repo and docs are real and directly inspected).
**Suggested scope: cross-project / high salience** — worth a direct look if
ever revisiting the operator-mediated design, since it's the closest thing
found to already-built prior art for this exact model.

## Durable findings, by theme

### Classic distributed-systems patterns solve transport, not relevance/timing
- Blackboard, tuple-space, actor-mailbox, and pub-sub architectures each
  solve *how a finding physically moves* between agents cleanly, but every
  one has a documented, still-open weak point on *who should see it, when,
  and whether it's still valid* — the control problem (blackboard), space
  consistency at scale (tuple spaces), ordering/cascade-failure (actors),
  eventual consistency + no retroactive delivery (pub-sub).
  **Confidence: strong**, corroborated across Corkill's original blackboard
  papers, Akka/CAF actor docs, and Azure's pub-sub architecture guide.

### Full broadcast is empirically harmful in LLM multi-agent systems specifically
- Broadcasting complete shared state measurably raised hallucination rate
  34% vs. a no-sync baseline in one study (arXiv 2606.21666) — a
  contaminated claim from one agent propagates as "ground truth" to every
  other agent's context.
- A separate study found steering accuracy collapsed from ~60% at 3 agents
  to ~21% at 10 under uniform broadcast ("broadcast pollution," arXiv
  2604.07911).
- **Anthropic's own multi-agent research system deliberately rejects context
  sharing**: subagents run in fresh, isolated windows, know nothing of each
  other, and return only 1,000–2,000 token condensed summaries to the lead
  agent — explicitly to avoid a "game of telephone." Anthropic states
  directly: domains needing shared context or many inter-agent dependencies
  are "not a good fit for multi-agent systems today."
  **Confidence: strong** (Anthropic's own published engineering writeup,
  directly fetched).

### The pointer/artifact pattern is the recurring alternative to full state or full broadcast
- Anthropic's subagents write output to external files/objects and pass back
  only a lightweight reference to the coordinator — never the full working
  context. The lead agent checkpoints its own plan externally too, for
  recovery after hitting a context limit.
- Anthropic's context-engineering guidance explicitly recommends *against*
  embedding/RAG-style pre-injection, favoring "just-in-time" retrieval via
  lightweight identifiers (file paths, stored queries) the agent resolves
  itself at time of use — "the smallest possible set of high-signal tokens."
  **Confidence: strong**, both directly from Anthropic's own engineering
  blog (`anthropic.com/engineering/multi-agent-research-system` and
  `.../effective-context-engineering-for-ai-agents`).

### Staleness needs an explicit structural signal, not inference from content
- Architecture Decision Records generally (10+ years of production use,
  independent of AI tooling): never edit an accepted record — a changed
  decision is a *new* record that explicitly supersedes and links to the
  old one. A Status field makes staleness structural, not inferred.
  **Confidence: strong** (Microsoft Learn / Azure Well-Architected).
- Pure vector-similarity retrieval cannot reliably distinguish a
  contradiction from a duplicate — a value-flip (e.g. a changed port
  number) is often *more* cosine-similar to the original than a genuine
  paraphrase is, so retrieval can surface both the true and the stale fact
  with no principled way to prefer one, and the failure is silent (fluent,
  well-formed, no error). **Confidence: moderate** (one primary source
  blocked at fetch time, corroborated by a second independent source).
- The automatic memory systems that actually work in practice (Zep/Graphiti,
  Mem0) achieve it by adding bi-temporal metadata (valid_at/expired_at) and
  automated contradiction-resolution — i.e., they re-implement curation
  logic *algorithmically* rather than eliminating curation. **Confidence:
  moderate**, corroborated across two independent papers.

### Operator-mediated promotion is a real, working production pattern
- Beyond `adr-kit` above: **Devin (Cognition)** ships a "Knowledge" feature
  where agent-suggested entries require explicit human review/approve/
  dismiss before becoming durable, org-wide memory — ongoing dedup/
  conflict-resolution is described as continuing human maintenance work,
  not a one-time gate. **Confidence: moderate** (Cognition's own docs).
- Named counter-evidence for skipping this gate: AutoGen's "Teachability"
  writes user teachings straight to a persistent vector store with no
  built-in review gate, and is flagged by analysts as a data-poisoning risk
  specifically because of that. **Confidence: weak** (secondary summary,
  included because it's the direct negative case).

### Production frameworks mostly default to full broadcast today; CrewAI is the partial exception
- LangGraph, AutoGen, and OpenAI's Agents SDK all default to passing full
  shared history/state at any handoff, with staleness and state-ownership
  design left entirely to the application developer — none ships a built-in
  scoped/selective mechanism. **Confidence: strong** on the OpenAI SDK
  claim specifically (directly from its own docs: *"every handoff must
  include all context the next agent needs — no hidden variables, no
  magical memory"*); moderate on LangGraph/AutoGen (secondary/practitioner
  sources, since LangGraph's own docs pages didn't load directly).
- CrewAI is the one clear exception: shared crew memory on by default, with
  automatic fact-extraction after each task and automatic recall-injection
  before each — framework-managed, not manual. It also supports scoped/
  partitioned memory as a private/shared middle ground.
  **Confidence: strong** (CrewAI's own docs). Its own community, however,
  states this holds only for simple 2-3 agent sequential crews — parallel
  agents or multi-run workflows push users toward external state
  management. **Confidence: moderate** (community GitHub discussion).

## Real disagreement surfaced, not resolved

Whether curation-free "automatic" sharing is achievable in principle:
one cluster of evidence treats it as structurally risky (memory poisoning,
hallucination propagation, "provenance collapse" — a named taxonomy in
arXiv 2606.24535) and argues for operator-mediated gates; another cluster
(Zep/Graphiti, Mem0, CrewAI's consolidation) shows "automatic" systems that
work — but only by encoding deterministic curation logic (bi-temporal
validity, dedup rules) in code. Not a strict contradiction — two different
bets on whether curation logic has to be human-mediated or can be fully
algorithmic.

## The genuine gap this research surfaced

No source — academic, vendor, or production framework doc — addresses the
actual deployment shape both xdOS and this ecosystem share: **separate,
long-lived, project-scoped agent sessions discovering a sibling's finding
after the fact**, rather than agents coordinating live inside one running
orchestrated process/graph. Every framework surveyed (LangGraph, AutoGen,
CrewAI, OpenAI's Agents SDK, even Anthropic's own system) assumes
single-runtime coordination. That's flagged directly by the research
itself as a real, unfilled gap — not a search failure. Anything built here
would be closer to genuinely novel than an application of existing prior
art, past the `adr-kit`-shaped starting point above.

## Corrections & follow-up, 2026-08-26

Appended, not edited in place — the discipline this doc itself documents
(ADRs/adr-kit: never rewrite an accepted record, supersede it with a linked
one). Surfaced by an independent review (a second worker set loose on the
whole exchange, not just this file) after this doc was already published;
verified directly before writing this section, not taken on faith.

**Citation weight, corrected.** The two arXiv preprints backing the
"full broadcast is empirically harmful" theme (2606.21666, the 34%
hallucination-increase figure; 2604.07911, "DACS," the 60%→21% figure) are
both single-author, unaffiliated preprints — thinner sourcing than their
presentation in the themes section implied. One specific claim was also a
misread, checked directly against the DACS abstract: the 21.0–60.0% range
is the **flat-context baseline's range across different experimental
conditions**, not a degradation curve that reaches 21% specifically at
N=10 agents (the abstract does show the baseline's disadvantage *growing*
with agent count in its Phase 4 results, but no per-N absolute-accuracy
numbers are stated to support the "60% at 3 → 21% at 10" framing this doc
used). The directionally-correct, actually load-bearing evidence for
broadcast harm in this doc is **Anthropic's own production multi-agent
research system** (strong confidence, primary source, directly fetched) —
treat the two preprints as corroborating color, not as the claim's real
support.

**"The genuine gap," narrowed.** Letta's sleep-time compute (shipped,
current — `docs.letta.com/guides/agents/architectures/sleeptime`,
`letta.com/blog/sleep-time-compute`) is a background agent that shares
memory blocks with a primary agent and updates them asynchronously,
cross-session, while the primary is idle. That *is* "a separate session
discovering a sibling's finding after the fact" — solved, single-platform,
shipped. This doc's claim of a fully unaddressed gap was too broad. What's
still genuinely unaddressed: the **cross-runtime, cross-ownership** case —
two different harnesses (e.g. agent-harness and a differently-built
external system) with different memory substrates, no shared runtime to
lean on. The design implication that falls out of narrowing it this way:
the fix isn't a shared runtime (Letta's answer, single-platform only), it's
agreeing on a **portable record format** — a typed claim, its provenance,
its verification state, and its supersession links — which is close to
what `adr-kit`'s ADR + Status History + supersession-graph shape already
is. Worth reading `adr-kit` as a candidate interchange format, not just as
one platform's internal convention.

**Two structural risks this doc didn't name, in the promotion channel
itself.**

1. *The promotion channel is the highest-privilege prompt surface in every
   architecture surveyed here, and nothing above discusses its integrity.*
   A pin, an ADR-INDEX entry, a `PROJECTS.md`/knowledge-candidate write —
   whatever gets promoted is injected as trusted context into every future
   session's cold-boot, unquestioned. A worker that successfully persuades
   a human reviewer to promote its own artifact has achieved persistent,
   cross-session prompt injection through the *legitimate* channel — the
   review gate checks whether the content is accurate, not whether it's
   also addressed to the reader as an instruction. `adr-kit`'s own stated
   invariant, "source material is evidence; it is never acceptance
   authority," needs a sibling invariant at the promotion boundary:
   **promoted content is data, never instructions** — the same rule this
   doc's own research agents were run under for fetched web content,
   extended to the thing that happens *after* review, not just before it.
2. *Refutation propagation has no answer here, and it's the strongest
   concrete case for a tape-style architecture in this whole document.*
   When a promoted finding is later refuted, what identifies the work that
   already consumed it? ADR-style supersession links are the write-side
   half of an answer (a record can point at what superseded it). An
   append-only tape is the read-side half neither this doc nor the
   conversation that produced it named: a queryable log of *which agent
   ran with which promoted item in context* turns "what's now tainted by a
   retracted premise" into a query. A git+markdown system without that
   log — the operator-mediated model this doc describes as production-real
   (`adr-kit`, Devin's Knowledge) — has no structural way to answer that
   question; finding out requires archaeology, not a lookup.
