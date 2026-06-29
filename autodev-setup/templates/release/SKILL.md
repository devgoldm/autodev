---
name: release
description: Ship a frozen release/* branch to production — owner-approved release/<version>→main PR with an authoritative security + engineering review, flag flips, post-release close watch on production logs, bug hunt, back-merge to develop, then a refactor pass. Use when the user runs /release, says "ship it", "release", or wants to get merged features into production.
---

# Release

Read `.claude/autodev/METHODOLOGY.md` and `.claude/autodev/config.json` first (and the selected presets under `.claude/autodev/presets/`). A release is one of the two human gates — never merge to production without explicit owner approval this session.

**Gitflow.** The model is `feature/<slug> → develop → release/<version> → main` with a `main → develop` back-merge after every ship (see METHODOLOGY § Branching). A release is **cut as a `release/<version>` branch off `develop`**, which **freezes the shipping set**: Build keeps landing features on `develop` without growing the open release PR, so the review at release-ask is **authoritative** (no re-baseline against a moving `develop` at ship time). In the autonomous loop, release-ask and ship run in the **`autodev-release`** scheduled task (daily, primary machine only).

## Steps

1. **Cut or detect the release branch.** `git checkout develop && git pull`. If no open `release/*`→`main` PR exists and `develop` is ahead of `main`, this is a **release-ask**: cut `release/<version>` off the current `develop` tip (default `release/YYYY-MM-DD`, suffix `-2`/`-3` for multiple per day), push it, and delete the local branch (the remote backs the PR). If an open `release/*`→`main` PR already exists, this is a **ship** — operate on that frozen branch; do **not** re-cut. Map the frozen set (`git log main..release/<version> --oneline`) to its work items/PRDs. Decide the **flag plan**: which flags flip ON in production with this release, which stay dark (skip the flag plan if `config.flags.mechanism` is `none`). Never touch platform-maintenance flags.

2. **Open the release PR + authoritative review** (release-ask). Open a PR (via the git-host adapter) with base `main` and head `release/<version>`, titled as a release with the change summary and a **## Flag plan** section. Run `/security-review` **and a general engineering review** (a subagent per `config.models.subagents` — regressions, logic bugs, edge cases, breaking schema/migration changes, perf, missing tests) over the **frozen** `main...release/<version>` diff. Because the branch is frozen this review is **authoritative** — post findings on the PR. Fix trivial, clearly-safe findings as commits **on the release branch**; surface the rest. Capture staging evidence (see step 3). Then notify "RELEASE PR #N READY — add `release:approved` to ship". **Stop here** until approved.

3. **Owner approval.** In the **autonomous loop** the gate is the **`release:approved` label** on the `release/*`→`main` PR — it approves the **frozen** snapshot reviewed in step 2. When run **interactively**, AskUserQuestion instead: present the frozen release contents + flag plan at the owner's technical level, and proceed only on approval; if they defer, stop cleanly. (Staging evidence: because flags ship ON in staging, the staging deploy runs the features — capture a short walkthrough + key screenshots per shipped issue into a LOCAL temp dir, display them inline in this run's transcript, and note on the PR that evidence is in the run; never commit or externally host it.)

4. **Ship — light delta check, then merge.** With `release:approved` set, the step-2 review stands. Only a **light delta check**: if any stabilization commits landed on the release branch since the review, re-run the affected review on just that delta. **On a release-blocking finding, HOLD:** remove the label, comment the blockers, open `ready-for-agent` bugfix work items targeting the release branch (git-host adapter), and notify "RELEASE HELD". Clean → merge via the git-host adapter using a **merge commit, never squash/rebase** — squashing into `main` makes it stop being an ancestor of `develop`.

5. **Deploy & flip.** Merging to `main` auto-deploys via the mechanism in `config.deploy`. Confirm the production deploy completed (check-runs / the deploy pipeline's build status / migrations job). Execute the flag plan (skip if `config.flags.mechanism` is `none`) **yourself via the flag API** (`config.flags` mechanism + the token in the host's gitignored env) — flip each shipped feature's production flag ON; do **not** AskUserQuestion for it (owner standing order). Then smoke-test production **observability-only** (no prod login per `config.prodTest`): confirm the flipped features are ON and the new endpoints respond (401/404 not 5xx), and watch the error rate.

6. **Close watch (~`monitoring.closeWatchMinutes` min):**
   - Poll production errors/exceptions every ~`monitoring.pollSeconds` seconds via the read-only monitoring credential (stay inside prompt-cache windows; use your harness's wakeup scheduling rather than busy-waiting).
   - Run one `/bug-hunt` pass during the window.
   - **Critical bug** (data loss, auth broken, feature dead, error spike): first contain it — flip the offending flag OFF yourself via the flag API (if `config.flags.mechanism` is `none`, go straight to the hotfix). If that doesn't contain it, hotfix — branch off `main`, fix via the build/verify subagent loop, PR to `main`, security review, merge, verify in prod, **back-merge to `develop`**.
   - **Non-critical**: open a `ready-for-agent` bugfix work item (git-host adapter) (or a `BACKLOG.md` entry where that is the tracker).

7. **Back-merge `main` → `develop`.** After a clean ship, open a `chore/sync-main-into-develop-<date>` PR and merge it with a **merge commit** so `develop` picks up any release-branch-only fixes and `main` stays an ancestor of `develop`. Delete the shipped `release/<version>` branch.

8. **Back off & close out.** Drop to the steady state in `config.monitoring` (default daily — the `autodev-bug-hunt` task covers daily triage). Shipped work items close automatically on the `main` merge (`Closes #N`); mark PRDs `shipped`; delete flags (and dead code) for features fully ON and stable — that can ride the refactor pass (no-op if `config.flags.mechanism` is `none`). (Where `BACKLOG.md` is the tracker, move released items to Done.)

9. **Refactor pass.** Every release pays the cleanup tax. If a human invoked `/release` interactively, run `/refactor-pass` now. In the **autonomous loop**, the `autodev-release` task runs the refactor pass itself right after a successful ship (see `ORCHESTRATION.md`).

10. **Report**: what shipped, prod health evidence from the watch window, and anything queued as follow-up.
