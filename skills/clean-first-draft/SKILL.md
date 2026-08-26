---
name: clean-first-draft
description: Use when rewriting or finalising a document (RFC, design doc, README, spec) that has accumulated revisions during the conversation. Produces a version that reads as if the reader is seeing it for the first time — no residue from prior corrections, no defensive framing, no traces of the discussion that shaped it.
---

# Clean first-draft rewrite

When the user asks for a doc to be rewritten "as a first draft" or "for a
new reader," they want the discussion residue stripped out. The reader of a
doc has not lived through the conversation. Anything that only makes sense
*because* of the conversation is noise to them.

## Tells of correction residue

Look for and remove:

- **Defensive framing.** Section titles or sentences that argue against an
  alternative the user once raised. Examples: *"Design strategy: reuse,
  don't fork"*, *"Why this satisfies the X constraint:"*, *"reuse-don't-fork
  is the whole point"*. Replace with neutral, descriptive titles.
- **Anti-framing on a field/decision.** Phrases like *"Cart state, **not**
  subscriber state"*, *"true iff X (rather than Y)"*. The first-draft reader
  has no Y in mind. Just say what it is.
- **"Excluded because" callouts.** Notes that explicitly call out what is
  *not* in the doc — *"min_spend is food-only and excluded"*, *"we don't
  need an applied/subscriber template here"*. If a thing isn't in scope,
  simply don't mention it.
- **Caps emphasis from arguments.** *"(UNCHANGED — single source of
  truth)"*, *"NO CHANGE"*, *"REUSED"*. Caps almost always mark a point that
  was contested. Lowercase or drop.
- **Resolved open questions.** Items in an "Open questions" list that the
  conversation already settled. Either drop them or fold the resolution
  into the body text.
- **Numbered-list gaps.** A list where item 3 was deleted, leaving
  `1, 2, 4, 5`. Renumber.
- **Stale cross-references.** *"(see §14, open question 3)"* where §14 has
  changed. Verify or remove.
- **Self-referential meta.** *"This document originally proposed X but..."*,
  *"After discussion we settled on..."*. Treat the chosen design as the
  design.
- **Justification by negation.** *"The complete payload is required because
  the client also renders X"* — fine if X is part of the spec, but drop it
  if the only reason for the sentence is to defend why a field exists at
  all.

## Process

1. **Don't edit in place.** Edits accumulate residue. Use Write to replace
   the file with a fresh version.
2. **Read the spec/requirements first**, not the prior draft. Build the
   doc's outline from the actual subject, not from the existing structure.
3. **Then read the prior draft** for facts, code references, and decisions
   you want to preserve. Treat it as raw material, not a template.
4. **Sweep for tells.** Grep the new draft for `not`, `unchanged`,
   `excluded`, `originally`, `however`, `but`, `(reuse`, `NO CHANGE`,
   parenthetical asides — each is a candidate for trimming.
5. **State decisions as decisions.** Each design choice is presented as
   the choice, not as the conclusion of a debate.

## What to keep

- Open questions that genuinely remain open.
- Justification when it informs trade-offs the reader still needs to
  evaluate (e.g. *"DB lookup chosen over PVS call to avoid the network
  round-trip"*).
- Pointers to existing code and config — those are facts, not residue.
- Comparisons to neighbouring systems when the reader needs that context
  (e.g. *"mirrors how Food Checkout's `subscription-cart` handles X"*).

## Heuristic

> If a sentence would confuse a reader who has never seen the prior
> conversation, it is residue. Cut it.
