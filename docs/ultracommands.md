# ultracommands

A user-space reproduction of Claude Code's built-in `/code-review ultra`
*observable behavior*, grown into a small multi-command framework — built
entirely from officially supported mechanisms (Skills + the Workflow
tool). It does not use or copy any proprietary implementation detail —
only mechanisms confirmed available in a real Claude Code installation.
Everything here is project-scoped: copy `.claude/skills/`, `scripts/`, and
`docs/` into a repo to install it, delete them to remove it. Nothing
touches global configuration.

## The commands

| Command | Job |
|---|---|
| `/ultraplan` | Scopes a vague idea into concrete goal(s), then chains the commands below when a task genuinely needs more than one phase |
| `/ultracode` | Adaptive, complexity-tiered implementation |
| `/ultra-review-local` | Multi-lens code review, reports only, never edits |
| `/ultradesign` | UI visual-layer (re)design — tokens/typography/iconography only, never logic |
| `/ultradebug` | Root-cause investigation, hands off to `/ultracode` for the fix |
| `/ultraretro` | Read-only analysis of a completed run for reusable lessons |
| `/ultraresearch` | Multi-angle research, synthesized into one sourced report |
| `/ultragoal` | Must-have completeness scoring against a category's real requirements |
| `/ultrabrainstorm` | Cheap, open-ended ideation across 3-5 blind angles |
| `/ultraonboard` | Retrofits an existing repo to this framework's own baseline hygiene |
| `/ultrainfra` | System-wide self-improving audit, gated by a reversibility rule |

Roughly a dozen other candidate commands were evaluated and deliberately
not built — most were just a task-framing of `/ultracode` (a flag
pretending to be a command), a couple duplicated what `/ultra-review-local`'s
existing dimension flags already cover.

## How one command actually runs

```
   /ultracode <task>                    (any command follows this shape)
        │
        ▼
 .claude/skills/<command>.md      — runs INLINE in the calling session,
        │                            not in a subagent, so triage and the
        │  triage: trivial/simple?    final report always stay under the
        │  → handle directly, done     calling session's own control
        │
        │  moderate or above? →
        ▼
 scripts/<command>.js  (Workflow tool — deterministic orchestration,
        │               not model-driven control flow)
        │
        ├─► plan        (persona: planner)
        ├─► implement   (persona: implementer — the only role with
        │                 write access to files)
        ├─► test        (persona: test-runner)
        ├─► verify      (persona: verifier — 4-state outcome:
        │                 confirmed / refuted / not_found / unable_to_verify)
        └─► fix → re-verify, looped as needed
        │
        ▼
 docs/knowledge.md pipeline
        │   routes durable learnings into the TARGET repo's own CLAUDE.md
        │   (project-tier) or Claude Code's own memory system
        │   (global-tier) — deliberately not a new parallel store.
        │   Always requires explicit human confirmation before writing.
        ▼
 final report to the calling session
```

A "specialist role" (planner, implementer, verifier, ...) is a plain
prompt-fragment document in `docs/personas/`, prepended to a built-in
agent type's prompt at dispatch time — not a custom subagent definition.
Custom subagent types (`.claude/agents/*.md`, invoked via a custom
`agentType`) were tried first and confirmed, by direct repeated testing,
not to resolve in this environment. The system was rebuilt around that
constraint rather than worked around it: `Explore`-typed roles (verifier,
test-runner, digest, every review lens) get a real, structurally enforced
guarantee — no `Edit`/`Write`/`Agent` tool access, confirmed by live test,
not just documented. `general-purpose` (the implementer, the only role
that writes files) has no structural block on spawning further agents —
a deliberate, stated residual risk, not an oversight.

## `/ultraplan` — chaining without becoming a flag

```
/ultraplan "vague idea"
        │
        ▼
  scope: clarifying questions → check what already exists vs. what's
         genuinely missing → one or more concrete goals
        │
        ▼
  shape classifier — how many phases does this actually need?
        A) exactly one  → decline to chain, point at that command directly
        B) diagnose → fix              (ultradebug → ultracode)
        C) fix → review                (ultracode → ultra-review-local)
        D) diagnose → fix → review     (all three, in sequence)
        │
        ▼
  ONE upfront approval for the whole chain
        │
        ▼
  runs phases straight through — but still stops wherever a phase's
  OWN procedure already requires a pause (a plan ready for approval,
  a blocked or contested verification result). Chain-level approval
  covers "run these phases," never "skip their internal gates."
        │
        ▼
  one consolidated report + one knowledge-candidate check,
  instead of one per phase
```

Shape A exists specifically as a guard against `/ultraplan` sliding back
into being a flag on `/ultracode` with extra steps — a goal that only
needs one phase gets refused, not humored.

## Verification, not vibes

The framework's core bet is that a finding (a bug, a completeness gap, a
root cause) isn't worth reporting until it's actually been checked against
the real code, not just reasoned about. Concretely:

- Verification outcomes are 4-state, not confirmed/refuted binary —
  `not_found` (the verifier never located what it was asked to check) is
  kept structurally distinct from `refuted` (checked and found nothing
  wrong) and excluded from every vote tally. Collapsing those two once
  produced a real false "contested" result during testing.
- Escalation to a multi-lens adversarial round is gated on evidence
  strength and security sensitivity, not on severity label alone — a
  strong, non-security finding stops after one verification pass instead
  of automatically paying for three more.
- Security sensitivity is an independent routing axis from complexity — a
  low-complexity but security-sensitive change still gets a mandatory
  dedicated verification pass it wouldn't get from complexity tier alone.

## Effort tuning

Every specialist role can run at a different reasoning-effort tier.
Mechanical roles (`digest`, `test-runner`) run at low effort. Roles doing
actual judgment calls — every `verifier` context, and the security
reviewer specifically — run at high effort. This was fitted, not assumed:
a fixed benchmark suite (`/ultrabenchmark`, a fixture corpus with known-
correct expected outcomes, run through the real scripts unchanged) is
what a change like this gets checked against, rather than trusted on
reasoning alone. As of 2026-09-01, CI checks the benchmark-results schema
itself and detects whether the Workflow tool is actually available in the
environment a run happened in, rather than assuming it — the benchmark
suite verifying its own inputs, not just the commands it exists to check.
`/ultrainfra` and `/ultraonboard` gained the same `id`/`status` tracking
mechanism the rest of the suite already had; real fixture coverage for
those two specifically is still open, not attempted alongside that fix.
