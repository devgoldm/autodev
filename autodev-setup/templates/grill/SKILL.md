---
name: grill
description: Grill a feature or design to the point of shared understanding — interview one question at a time with a recommended answer, challenge every decision against VISION.md, CONTEXT.md, and the ADRs, walk the whole decision tree until nothing is left open, and update CONTEXT.md/ADRs inline as decisions crystallise. Bundled with autodev and used by /vision; invoke directly when you want to stress-test a plan before building.
---

# Grill

A disciplined interview that turns a fuzzy idea into a fully-decided design the owner genuinely understands. It is autodev's own grilling skill — `/vision` calls it, and you can run it any time you want to pin down a plan before writing code. It depends on nothing outside this project.

Read `.claude/autodev/METHODOLOGY.md` and `.claude/autodev/config.json` first so you grill at the owner's recorded technical level. Load `VISION.md`, `CONTEXT.md`, and `docs/adr/` so every question is grounded in what the product already is and has already decided.

## Principles

- **One question at a time.** Never batch. Each question gets the owner's full attention and its answer shapes the next question.
- **Always carry a recommendation.** Every question comes with *your* recommended answer and a one-line why. The owner can accept, refine, or reject — but they're never staring at a blank prompt. Use AskUserQuestion so the recommended option is first and labelled.
- **Walk the whole tree.** A design is a tree of decisions; a decision opens sub-decisions. Keep going depth-first until every branch terminates in something concrete. "We'll figure it out later" is a branch left open — name it as a deliberate deferral or resolve it.
- **Challenge against the docs.** Test each answer against `VISION.md` (does it serve the stated intent and respect the non-goals?), `CONTEXT.md` (does it use the domain's existing words correctly, or introduce a new one that needs defining?), and `docs/adr/` (does it contradict a recorded decision? then either this changes or that ADR does — in writing).
- **Calibrate to the owner.** For technical owners, grill on implementation trade-offs, interfaces, schema, failure modes. For non-technical owners, grill on behavior, cost, edge cases, and risk in plain language. Either way the owner must end up able to explain how it works, at their level.

## What to grill (cover every one that applies)

1. **Problem & user** — who exactly is this for, what is the problem in their words, how do we know it's real.
2. **Scope boundary** — what's explicitly in and out for this unit of work. The out-list is the most valuable output of the session.
3. **Behavior** — the precise user-visible behavior, including the unhappy paths: empty states, errors, permissions, concurrency, limits.
4. **Domain language** — every noun/verb that's load-bearing. If a term is new, define it; if it overlaps an existing `CONTEXT.md` term, reconcile them.
5. **Data & interfaces** — what state changes, what the contracts are (API shape, schema, events). Capture the *decision*, not file paths.
6. **Failure & reversibility** — what happens when it breaks, how it's contained (the flag), how it's rolled back.
7. **Acceptance** — how we'll know it works: the concrete things that must be observably true.

## Output

As decisions crystallise, **update the docs inline** (this is the point — don't leave it for later):

- **`CONTEXT.md`** — add or sharpen any term the grilling settled. Resolve flagged ambiguities.
- **`docs/adr/NNNN-slug.md`** — write an ADR for any decision that is hard to reverse, surprising without context, and a genuine trade-off. Record the alternatives considered and why this one won.
- **`VISION.md`** — if the grilling clarified or shifted product direction, propose a diff and show the owner before applying (the owner owns this file).

End the session with a crisp recap: the decisions made, the boundary (in/out), the open deferrals (named, not hidden), and the acceptance criteria. When invoked by `/vision`, that recap feeds straight into the PRD.

## Rules

- Don't start implementing — grilling produces decisions and documents, nothing else.
- Don't let a contradiction with `VISION.md`/an ADR pass silently. Surface it; resolve it in writing one way or the other.
- Stop when the tree is fully walked, not when you're tired — but if the owner is fading, checkpoint the decided branches into the docs and name what's still open so the next session resumes cleanly.
