---
name: vision
description: Turn VISION.md into the next concrete feature — propose candidates, grill the owner on the chosen design, write a PRD with atomic tasks, and queue it as a ready-for-agent work item (the work tracker; or BACKLOG.md as a fallback — see ORCHESTRATION.md). Use when the user runs /vision, asks "what should we build next", proposes a feature idea, or wants to plan new work from the vision.
---

# Vision → Feature

Read `.claude/autodev/METHODOLOGY.md` and `.claude/autodev/config.json` first. Communicate at the owner's recorded technical level throughout.

## Steps

1. **Load state**: `VISION.md`, `CONTEXT.md`, the open work tracker (list it via the git-host adapter — or `BACKLOG.md` where that is the tracker), and skim recent `docs/prds/`. Note what's shipped, in progress, and parked.

2. **Choose the feature**:
   - If the owner came with a specific feature request, use that.
   - Otherwise, derive 2–3 candidate features that most advance the vision from its current state (favor vertical slices a user would notice over infrastructure). Present them via AskUserQuestion with a recommendation and a one-line "why now" each. The owner picks.

3. **Grill the design** with autodev's bundled `grill` skill (`.claude/skills/grill/` — installed with this project): it interviews one question at a time with a recommended answer, challenges the plan against `VISION.md`/`CONTEXT.md`/ADRs, walks every branch of the design tree until there are no open decisions, and updates `CONTEXT.md`/ADRs inline. Calibrate depth to `owner.technicalLevel` — implementation trade-offs for technical owners; behavior, cost, and risk in plain language for non-technical owners. The owner must end up genuinely understanding how the feature will be built, at their level.

4. **Write the PRD** to `docs/prds/NNNN-slug.md` (next number) using `.claude/autodev/PRD-FORMAT.md`. Include:
   - The atomic task checklist: small, independently verifiable tasks a single subagent can complete and verify in one sitting. Order them as tracer-bullet vertical slices (thin end-to-end first, then flesh out).
   - The feature's flag name (omit if `config.flags.mechanism` is `none`).
   - Testing decisions — every feature ships with tests.

5. **Queue it**: open a `ready-for-agent` work item linking the PRD, with a `priority:` label (the orchestrator picks by priority label, then oldest-created) — see the git-host adapter. Where `BACKLOG.md` is the tracker instead, add it under Ready, linking the PRD.

6. **Reflect into the docs**:
   - If the grilling clarified or changed the product direction, propose a `VISION.md` edit — show the owner the diff before applying (the owner owns that file).
   - Update `CONTEXT.md` with any new/sharpened terms; write ADRs for decisions that are hard to reverse, surprising, and a real trade-off.

7. **Hand off**: tell the owner the feature is queued and they can start the build with `/build-next`.

## Rules

- One feature per session — depth over breadth.
- Never start implementing here. This command produces decisions and documents only.
- If the owner's request conflicts with VISION.md, surface the conflict explicitly — either the request or the vision should change, in writing.
