---
name: brainstorming
description: "Use before drafting a PLAN.md whenever the source RFC leaves an implementation-level decision open — sequencing, migration strategy, rollout approach, testing strategy, or which of several valid task breakdowns to use. Resolves these through one-at-a-time dialogue before any plan is written."
---

# Resolving Plan Decisions

Turn the open, implementation-shaping questions an RFC leaves behind into settled decisions, through natural one-at-a-time dialogue, before `writing-plans` drafts anything.

This is a trimmed, plan-specific variant: it resolves *implementation* decisions (how to sequence, migrate, roll out, and break down work), not product design — the product design is already settled in the RFC by the time this runs.

<HARD-GATE>
Do NOT invoke `writing-plans` or write any part of a PLAN.md until every blocking decision below has been presented and the user has answered it. This applies even when the RFC looks fully specified — an RFC can be complete on contracts and flows while still leaving sequencing or migration strategy open.
</HARD-GATE>

## When to Use

- Stage 1 of the rfc-to-plan pipeline, right before `writing-plans`.
- The RFC has an "Open Questions" section, or its Key Decisions don't extend to implementation order/rollout.
- A multi-repo change where the cross-repo landing order isn't dictated by the RFC alone.

Not for: resolving product/design questions — those belong to the RFC and its own review cycle. If a question here would actually change a contract or flow the RFC defines, that's a sign the RFC itself is incomplete; surface it as a blocking gap for the human rather than deciding it here.

## What Counts as a Plan Decision

Only implementation-shaping choices that affect task boundaries or sequencing, for example:

- **Migration strategy** — backfill-then-cutover vs dual-write vs big-bang, when the RFC specifies the end state but not the path there.
- **Rollout approach** — feature flag vs staged repo-by-repo release vs all-at-once, when the RFC doesn't already mandate one.
- **Cross-repo landing order** — which service's change must merge first when the RFC defines a shared contract but not who ships first (multi-repo mode; the command passes this skill the resolved service list precisely so this decision has something concrete to reason about).
- **Task granularity** — where a single RFC element is large enough that it could reasonably be one task or several, and the split affects review/rollback size.
- **Test strategy** — whether a change needs new integration tests, a migration dry-run, or is adequately covered by existing suites.

If none of these apply — the RFC and its coverage matrix already fully determine sequencing — skip this skill and go straight to `writing-plans`.

## Process

1. **Read the RFC (and PRD if provided) in full** before asking anything — most "open" questions turn out to have an implied answer once the whole document is read.
2. **List the genuine decisions** using the categories above. Don't manufacture a decision where the RFC already implies one.
3. **Ask one question at a time.** Prefer multiple choice with a recommended default; open-ended is fine when options don't apply. Only one question per message.
4. **Lead with a recommendation and reasoning** for each, rather than presenting a bare list of options.
5. **Record settled decisions** as a short list — these feed directly into the plan's "Decisions" section; no separate design doc is written for this stage.

## Key Principles

- **One question at a time** — don't overwhelm with multiple questions in one message.
- **Multiple choice preferred**, with a stated recommendation.
- **Don't re-litigate the RFC** — if a question would change a contract or flow, it belongs back with the RFC's author/reviewer, not resolved here.
- **Stop when the categories above are exhausted** — this is a short, bounded pass, not a full design conversation.

## Handoff

Once every blocking decision is answered, pass the settled list directly into `writing-plans` — do not write an intermediate spec document. The terminal state of this skill is invoking `writing-plans`.
