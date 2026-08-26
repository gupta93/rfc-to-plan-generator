# rfc-to-plan

A Claude Code plugin that turns an approved RFC (plus an optional PRD) into a repo-grounded implementation plan through a four-stage pipeline: resolve any implementation-level decisions the RFC leaves open, draft the plan, clean it, then verify it against the RFC and refine until it either fully covers the RFC or is blocked only by decisions a human must make.

This is the natural next stage after [`prd-to-rfc`](../prd-to-rfc-plugin): that plugin takes a PRD to an RFC; this one takes the RFC the rest of the way to a task list ready for execution. The two are separate, independently installable plugins — this one only reads the RFC (and optionally the PRD) as input files, it doesn't depend on the other plugin being installed.

## What it does

Running `/rfc-to-plan <path-to-rfc> [path-to-prd]` drives these stages, pausing for your review after each of the first three:

1. **Resolve plan decisions** — identifies implementation-shaping open questions the RFC leaves behind (migration strategy, rollout approach, cross-repo landing order, task granularity, test strategy) and resolves each one at a time. If the RFC already determines everything, this stage is a no-op.
2. **Draft** — first reads each target repo's own `CLAUDE.md`/`AGENTS.md`/`SKILLS.md`/`README.md` for its conventions and classifies its service type, then writes a `PLAN.md` that decomposes the RFC into concrete, dependency-ordered, additive-by-default tasks, each grounded in the current code it changes (`file:line`), each addressing the non-functional concerns its service type calls for, and each carrying its own Given/When/Then test-case table. Anything the skill can't ground in the RFC/PRD/decisions/repo evidence becomes a Clarifying Question rather than a guess, and a task list too large to read as one document gets split into numbered slices with an index.
3. **Clean** — rewrites the plan so it reads fresh, with no residue from the decision dialogue that shaped it.
4. **Verify & refine** — runs a cheap structural check, then checks four coverage dimensions (RFC-element, non-functional, per-task test cases, and whether every Clarifying Question is answered), auto-closes gaps it can resolve from the codebase for up to 3 rounds, and stops to ask you only when a gap needs a decision, a question needs an answer, the round cap is hit, or coverage is complete.

All documents are written locally under `docs/` and are never committed.

## Any language, any service type

The repos this plugin plans against aren't assumed to be one language or one shape. Before decomposing tasks, `writing-plans`:

- **Reads the repo's own conventions** — `CLAUDE.md`, `AGENTS.md`, `SKILLS.md`, `README.md`, `CONTRIBUTING.md` — for its stack, test framework, and architectural patterns, rather than assuming a generic default.
- **Classifies the repo's service type** — api-service, worker/consumer, frontend-web, mobile, batch/cron, library/sdk, or unknown — from stated descriptions in those files first, or repo structure signals otherwise (see `skills/writing-plans/service-playbooks.md`).
- **Applies that type's own non-functional checklist**, so the plan asks the right questions for what the service actually is:
  - **api-service** — idempotency, failure tolerance/retries, DB consistency, latency, concurrency, circuit breaking, observability, caching.
  - **worker/consumer** — consumer idempotency and dedup, retry policy, dead-letter/skip handling, ordering guarantees, backpressure, checkpointing.
  - **frontend-web / mobile** — state management, offline/error/loading UX, accessibility, platform conventions, client-side caching, analytics. Backend concerns like DB consistency and circuit breakers are explicitly *not* applied here — a client-facing repo is planned, not skipped, but against its own checklist.
  - **batch/cron** and **library/sdk** get their own shorter checklists; an **unknown** type gets a minimal generic one and is flagged for you to confirm at the Stage 2 checkpoint.

`plan-coverage-check` verifies these dimensions: every RFC element traces to a task, every applicable non-functional concern is either addressed by a task or deliberately excluded with a stated reason, and every task's test cases cover what its service type requires — a plan that covers every endpoint but forgets a worker's dedup logic, or a task whose only test case is the happy path, is not treated as complete.

## No assumptions — ungrounded facts become questions, not guesses

Every detail in a plan traces to the RFC, the PRD, a Stage 1 settled decision, or evidence read from the repo. When none of those settle something the plan needs — an unprecedented convention, an ambiguous RFC phrase, a choice between two equally-plausible task boundaries — `writing-plans` does not invent a default. It adds the item to the plan's **Clarifying Questions** section as a direct, answerable checklist item, and keeps decomposing everything that doesn't depend on the answer.

This section is deliberately not the same thing as an RFC's or PRD's "Open Questions." If answering something would change a contract, a flow, or a decision the RFC is supposed to own, that's a gap in the RFC to raise — not a question the plan quietly holds instead. A Clarifying Question here is scoped to this plan's own task boundaries, sequencing, or conventions: things the RFC was never responsible for settling. `plan-coverage-check` treats every unanswered Clarifying Question as a blocking stop condition — it's never auto-refined, because by definition nothing in the plan's available sources could resolve it; only the human can.

## Code-change discipline

Every task is additive by default — its `Removes` field is "none" unless the RFC explicitly designates that exact code for removal or replacement, and that was part of what the human approved when reviewing the RFC. `writing-plans` never decides on its own, during planning, to delete or rewrite something the RFC never mentioned touching. Task descriptions are also expected to respect the surrounding code's existing patterns rather than reinvent them: Single Responsibility (don't bundle unrelated behaviors into one task just because they touch the same file), DRY (reuse an existing validator/helper instead of describing a new copy), and KISS (the smallest change that satisfies the RFC element, not more generality than was asked for).

## Splitting large plans

A single `PLAN.md` stops being useful once it stops being readable in one sitting. Rather than stuff a large requirement into one long file, `writing-plans` splits at a natural seam — a phase boundary, a milestone, a cluster of tasks with no cross-dependency on another cluster — once a service's task list passes roughly 12–15 tasks or the RFC itself describes distinct phases. The result is numbered slice files (`docs/YYYY-MM-DD-<topic>-plan-01-<slice-name>.md`, `-plan-02-...`) plus a short index, each slice a complete, self-contained plan on its own. This composes with multi-repo mode: any single service's child plan can itself be split the same way if its own task list is large. `plan-coverage-check` checks coverage across the full set of slices as one unit, not file by file.

## TDD-ready plans

"Verify: add a test" isn't specific enough to hand to a developer or an executing agent without them having to invent the actual cases — which reintroduces the ambiguity a plan exists to remove. Every task in a `PLAN.md` produced by this plugin carries a Given/When/Then test-case table instead:

| ID | Category | Given | When | Then |
|---|---|---|---|---|
| TC1 | happy | a valid refund request for an active order | `POST /v1/refunds` called | 200, refund created |
| TC2 | edge | request amount equals the order's remaining balance | same endpoint called | 200, order balance goes to 0 |
| TC3 | error | request for an order that doesn't exist | same endpoint called | 404, no refund record created |
| TC4 | idempotency | the same request sent twice | endpoint called twice | second call returns the first result, no duplicate side effect |

Every task requires at least a happy, an edge, and an error case, plus whatever categories its service-type playbook adds (an idempotency case for a side-effecting api-service task, a dedup case for a worker, an offline/error-state case for a frontend/mobile task — full list in `service-playbooks.md`). These are specific enough to write as failing tests before the implementation exists — real TDD, not just a reminder to test. `plan-coverage-check` checks this as its own coverage dimension, and a placeholder case (`...`, "TBD") counts as missing, not covered.

## Guardrails

Stage 4 is the one part of this pipeline that iterates without a human turn between rounds, which is exactly where agentic loops tend to go wrong elsewhere — unbounded retries, scope creep beyond intended boundaries, output nobody checked before it was acted on. `GUARDRAILS.md` at the plugin root maps the standard input/output/tool-action/runtime/ops guardrail layers onto what this plugin actually is (a file-scoped pipeline, not a tool-calling production agent) and states what's enforced at each layer. In short:

- Every exploration agent is read-only; every skill's write access is scoped to `docs/`; nothing in this pipeline ever runs a git commit.
- Every task is additive by default; nothing is removed or rewritten unless the RFC explicitly says so and a human approved it.
- Nothing is assumed; an ungrounded fact becomes a Clarifying Question, never a guess.
- Stage 4's refine loop is capped at 3 rounds and stops immediately if a round makes zero progress, rather than looping to the cap regardless — and never auto-answers a Clarifying Question, which always waits for the human.
- Every generated document records the commit SHA(s) it was grounded against and a changelog of what each refinement round changed, since there's no separate logging layer to check instead.
- `plan-coverage-check` never trusts `writing-plans`' own claim of completeness — it independently re-derives coverage every round (the executor/validator pattern for multi-agent handoffs).

## Contents

```
rfc-to-plan-plugin/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest
│   └── marketplace.json     # local marketplace entry
├── commands/
│   └── rfc-to-plan.md       # the /rfc-to-plan orchestrator command
└── skills/
    ├── writing-plans/
    │   ├── SKILL.md                # RFC (+ optional PRD) -> PLAN.md, new for this plugin
    │   └── service-playbooks.md    # classification + non-functional checklist per service type
    ├── plan-coverage-check/ # verifies a plan covers its RFC and its non-functional concerns, refines, new for this plugin
    ├── brainstorming/       # bundled dependency, trimmed to plan-level decisions only
    └── clean-first-draft/   # bundled dependency, vendored unchanged
```

The two bundled skills are vendored so the plugin is self-contained; the command invokes them by their plugin-namespaced names (`rfc-to-plan:brainstorming`, `rfc-to-plan:clean-first-draft`), so they do not collide with any same-named skills installed elsewhere. Unlike the `prd-to-rfc` plugin, `brainstorming` here is a trimmed, plan-specific variant — it resolves implementation decisions (sequencing, migration, rollout, granularity), not product design, and hands off directly to `writing-plans` rather than writing an intermediate spec document.

## Install

From within Claude Code, register this directory as a local marketplace and install the plugin:

```
/plugin marketplace add /<path-to-dir>/rfc-to-plan-plugin
/plugin install rfc-to-plan@rfc-to-plan-dev
```

Restart the session so the new command and skills register.

To confirm it loaded, check that `/rfc-to-plan` appears in the command list and that the skills are listed as available.

## Use

```
/rfc-to-plan docs/2026-07-10-refunds-rfc.md docs/2026-07-01-refunds-prd.md
```

The PRD argument is optional — omit it if the RFC alone is enough context. The command runs the four stages above. At each checkpoint (after Stage 1, 2, and 3) it presents the artifact and waits for you to approve or request changes. In Stage 4 it iterates on its own, and returns to you only with either a fully covered plan or a short list of blocking decisions.

The two new skills also work on their own, outside the pipeline:

- Ask to turn an RFC into tasks, or "how do we actually build this", to trigger `writing-plans`.
- Ask whether an existing plan covers its RFC, or to check a plan for gaps, to trigger `plan-coverage-check`.

## Multi-repo mode

When the RFC's design spans several backend services, place an `rfc-to-plan.json` file at the root of the working directory listing each service and its local checkout path:

```json
{
  "services": [
    { "name": "checkout-api", "path": "/Users/you/dev/checkout-api", "description": "cart + checkout orchestration" },
    { "name": "payments",     "path": "../payments" }
  ]
}
```

- `name` — short unique identifier, used in child-plan filenames and the coverage matrix.
- `path` — absolute, or relative to the working directory; must be an existing local checkout.
- `description` — optional one-line hint about what the service owns, used to route tasks to the right repo.

This is a separate config file from `prd-to-rfc.json` (used by the earlier pipeline stage) — the service list relevant to execution sequencing isn't assumed to be identical to the one used for gap analysis, though the two will often list the same services.

With the file present, the pipeline runs in multi-repo mode: `writing-plans` emits a **master plan** (`docs/<date>-<topic>-plan.md`) plus one **child plan per affected service** (`docs/<date>-<topic>-<service-name>-plan.md`). The master owns the cross-repo dependency graph and any shared-contract tasks; each child owns its service's task list. The coverage matrix in the master traces every RFC element to the service and task that implements it, applying a multi-owner rule: a requirement split across services is only Covered once every owning service's task exists.

With no `rfc-to-plan.json` present, the plugin runs in single-repo mode exactly as described above — one codebase, one plan.

All documents are written locally under `docs/` and are never committed.

## Requirements

- Claude Code with plugin support.
- An RFC file to plan from, and the repository (or repositories, in multi-repo mode) the plan targets open as/reachable from the working directory.
- The ability to dispatch exploration subagents (used to ground tasks in current code) and, for PDF PRDs, PDF reading.
