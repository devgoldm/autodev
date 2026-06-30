---
name: refactor-pass
description: Post-release conservative cleanup — reduce complexity so agents can navigate the codebase, delete dead flags/code, refresh CONTEXT.md and ADRs, behavior-preserving and test-green throughout. Use when the user runs /refactor-pass, after a release, or asks to clean up / simplify the codebase.
---

# Refactor Pass

Read `.claude/autodev/METHODOLOGY.md` and `.claude/autodev/config.json` first (and the selected presets under `.claude/autodev/presets/`). Runs after every release. The goal is **AI legibility**: the codebase must stay simple enough that a cold agent session can navigate and extend it. Behavior-preserving only.

## Steps

1. **Branch**: `feature/refactor-<date>` off fresh `develop`. Confirm the full test suite is green before touching anything — that's the baseline.

2. **Find targets.** Read `CONTEXT.md` and `docs/adr/` for the domain language and load-bearing decisions, then run the bundled **`thermo-nuclear-code-quality-review`** skill (`.claude/skills/thermo-nuclear-code-quality-review/`) over the code that changed since the last refactor pass (diff `develop` against the previous `feature/refactor-<date>` tag/branch, or the whole module if that's unclear). Use its standards as the lens for what counts as a target — file-size blowups, spaghetti-condition growth, thin/“magic” abstractions, boundary leaks, and "code judo" simplifications. Combine its findings with the priority list below:
   - Dead code: flags fully released and stable → delete the OFF code path, then delete the flag itself via the flag API (code-first; owner standing order to manage flags via the API, not AskUserQuestion; skip the flag deletion if `config.flags.mechanism` is `none`); unused exports, unreachable branches.
   - Duplication introduced by recent features (subagents working independently tend to re-create helpers).
   - Naming drift from `CONTEXT.md` language — rename code to match the domain terms.
   - Oversized files/modules and tangled coupling that make navigation expensive.

   The thermo-nuclear skill is **ambitious** by design (it pushes for restructurings that delete whole branches/layers); the refactor pass stays **behavior-preserving**. So treat each thermo-nuclear finding as a candidate target, then apply the § Rules gate in step 3: take the ones whose tests make a behavior-preserving fix safe, and icebox the rest as work items rather than reaching for behavior changes.

3. **Refactor conservatively** via subagents (serial, same branch, per METHODOLOGY; pinned to `config.models.subagents`): each subagent gets one target, must keep behavior identical, and must keep the test suite green — run it before and after. Skip any target whose refactor would change behavior or whose tests don't cover it well enough to be safe; note those as `icebox` work items (or in `BACKLOG.md` Icebox) instead.

4. **Refresh the docs**: update `CONTEXT.md` (new terms, resolved ambiguities), write ADRs for any structural decisions made, tidy stale work items/labels (or prune `BACKLOG.md`'s Done section where that is the tracker), fix stale statements in README/CLAUDE.md.

5. **Ship it like any change**: PR into `develop` (via the git-host adapter), `/security-review`, fix findings, merge once clean. No owner approval needed.

6. **Report**: what was simplified, what was deleted (especially dead flags), what was deliberately skipped and why.

## Rules

- Never bundle behavior changes or "small improvements" into a refactor — those go through `/vision` → `/build-next`.
- If tests are too thin to refactor safely, the deliverable becomes *adding tests* (still behavior-preserving) and the structural work waits for the next pass.
