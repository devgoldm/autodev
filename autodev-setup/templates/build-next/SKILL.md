---
name: build-next
description: Build the top ready-for-agent work item from the work tracker (host issues by default; or BACKLOG.md as a fallback — see ORCHESTRATION.md) — feature branch off develop, subagents implement and verify each atomic task serially, whole-feature verification, security-reviewed PR into develop, docs updated. Use when the user runs /build-next, says "build the next feature", or wants to start/resume building a queued feature.
---

# Build Next Feature

Read `.claude/autodev/METHODOLOGY.md` and `.claude/autodev/config.json` first (and the selected presets under `.claude/autodev/presets/`). You are the orchestrator: you plan, spawn subagents for labor (pinned to `config.models.subagents`), verify, and escalate. You do not hand-write the feature yourself.

## Steps

1. **Pick up the work**: the top `ready-for-agent` work item (the work tracker — see the git-host adapter — sorted by `priority:` label then oldest-created; or the feature the owner named; or resume an `in-progress` work item — the PRD's task checklist records where it stopped). Read its linked PRD fully. Label the work item `in-progress`. (Where `BACKLOG.md` is the tracker, take the top `Ready` item and move it to In progress instead.)

2. **Branch**: `git checkout develop && git pull`, then `git checkout -b feature/<slug>`. Everything happens on this one branch — no worktrees, no parallel mutating agents.

3. **Flag first** (skip this step entirely if `config.flags.mechanism` is `none` — the methodology supports a no-flags mode): create the feature's flag (name is in the PRD frontmatter) **yourself via the flag API** (`config.flags` mechanism + the token in the host's gitignored env) — **OFF in production, ON in preview/staging** (`config.flags.stagingOnProdOff`). Do **not** AskUserQuestion for it (owner standing order: manage flags via the API). Some projects enforce staging-on/prod-off **in code**, in which case the flag only needs to *exist* off. Verify it evaluates via the project's flags module / flags endpoint. All new behavior must be invisible and harmless when the flag is off.

4. **Build tasks serially.** For each unchecked task in the PRD, spawn one subagent pinned to the model in `config.models.subagents` with a self-contained prompt containing: the PRD path, the task text, the flag name, relevant file paths, the project's verify commands (test/typecheck/lint), and these instructions:
   - Implement the task on the current branch, in place.
   - Add or extend tests covering the task.
   - Run the verify commands and actually exercise the change; fix and re-verify in a loop until everything passes.
   - Commit with a descriptive message and report what was done, how it was verified, and anything surprising.

   After each subagent returns, sanity-check its report and the diff, tick the task in the PRD, and update the Build log. Then the next task.

5. **Whole-feature verification** — same loop shape, one level up:
   - Spawn a verify-subagent: run the full test suite, run the app (flag ON locally), and exercise the feature end-to-end like a user; report pass/fail with evidence.
   - For each failure, spawn a fix-subagent (self-contained prompt with the failure evidence), then re-verify.
   - **3 strikes**: after `escalation.maxFixCycles` failed fix→verify cycles, stop. Park the branch, set PRD status `parked` with a Build log write-up of what was tried, label the work item `parked` with a note (or move the `BACKLOG.md` item to Icebox), and AskUserQuestion the owner: simplify scope / different approach / drop it.

6. **PR into develop**: push the branch, open a PR (via the git-host adapter) targeting `develop` with a summary linking the PRD and a `Closes #N` line for the work item. Then run `/security-review` on the branch and fix every finding in this same PR (fix-subagents, re-review until clean).

7. **Merge autonomously** once verification and security review pass (squash merge, delete branch). No owner approval needed at this gate.

8. **Close out the docs**: PRD status `built`, the work item moves to `in-review` and stays open until released — the PR's `Closes #N` closes it on the release merge (where `BACKLOG.md` is the tracker, the item stays In progress, noting "merged to develop"), update `CONTEXT.md`/ADRs if implementation crystallised anything new. Confirm the develop build succeeded via the deploy pipeline's build status (see the stack preset) (read the build logs if it failed — a red preview build blocks the next release) and that the preview deployment is healthy.

9. **Report** to the owner at their technical level: what was built, how it was verified, that it's flag-off on develop, and that `/release` ships it.
