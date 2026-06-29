---
name: autodev-release
description: Autodev RELEASE loop (primary machine only) — Gitflow ship/release-ask on a frozen release/* branch, then post-release close-watch + back-merge + refactor.
---

<!-- TEMPLATE. Copy to ~/.claude/scheduled-tasks/autodev-release/SKILL.md on the PRIMARY machine ONLY
     and replace every <PLACEHOLDER> with this project's values (from .claude/autodev/config.json):
       <REPO_PATH>            absolute path to the live checkout
       <REPO_SLUG>            owner/repo for gh
       <INTEGRATION_BRANCH>   default: develop
       <PRODUCTION_BRANCH>    default: main
       <RELEASE_PREFIX>       default: release/  (version: release/YYYY-MM-DD, suffix -2/-3)
       <ENV_SOURCE>           how to load gitignored env (read-only monitoring + flag + staging-access creds)
       <STAGING_URL>          e.g. https://staging.example.com (omit the staging-evidence step if none)
       <STAGING_HEALTH>       how to confirm the latest integration deploy is green (CI run / check-runs)
       <DEPLOY_CHECK>         how to confirm the production deploy completed (CI run / check-runs)
       <FLAG_FLIP_NOTES>      how to flip a production flag yourself (REST base + app id + token var), or
                              the project's flag mechanism; remove if the project has no flags
       <TEST_CMD>/<BUILD_CMD> the project's verify commands
     Schedule DAILY at a fixed local time (e.g. `45 9 * * *`; cron follows the clock through DST),
     notifyOnCompletion:false (self-notifies on release-ready/escalation). Keep the HARD INVARIANT intact.
     Commands below use the GitHub `gh` adapter as the reference; if `config.gitHost` is another host,
     swap them for that adapter's equivalents (see `.claude/autodev/presets/`).
     (`PushNotification` is the Claude Code runtime primitive — under another loop runtime, use that
     runtime's notify channel per `config.loop.notify`; see the runtime preset.)
     (The `release:approved` label is the reference approval signal; see `config.loop.approval` +
     the runtime preset.) -->

You are the autodev RELEASE loop, running as a DAILY scheduled task on the PRIMARY machine ONLY (it owns prod monitoring + is the owner's approval gate). DO NOT run this loop on secondary build machines. Each run performs AT MOST ONE of Ship / Release-ask; after a successful Ship it also runs prod close-watch + a Gitflow back-merge + a refactor pass. Idempotent — GitHub labels and the open release PR act as locks.

GITFLOW: `feature/* → <INTEGRATION_BRANCH> → <RELEASE_PREFIX><version> → <PRODUCTION_BRANCH> (production)`, with a back-merge `<PRODUCTION_BRANCH> → <INTEGRATION_BRANCH>` after every ship (and hotfixes off production). The `<RELEASE_PREFIX><version>` branch FREEZES the shipping set: Build keeps merging features into the integration branch with NO effect on an open release PR, so the Release-ask review is AUTHORITATIVE (no re-baseline at Ship).

REPO (owner's LIVE checkout — treat as READ-ONLY base): `<REPO_PATH>`
GitHub repo for `gh`/issues/PRs: `<REPO_SLUG>`
HARD INVARIANT: never switch the main checkout's branch, edit its working tree, or run installs in it. Cutting/merging release branches is done via `gh` and remote refs (`git branch <name> origin/<INTEGRATION_BRANCH>` only creates a ref — it does NOT switch the working tree); any code work happens in an ephemeral worktree under `.claude/worktrees/` removed before finishing.

SETUP EACH RUN:
1. `cd <REPO_PATH>` and run `git fetch origin --prune`. Do NOT change its checked-out branch.
2. Read `.claude/autodev/METHODOLOGY.md`, `.claude/autodev/ORCHESTRATION.md`, and `.claude/autodev/config.json`.
3. `<ENV_SOURCE>` for the read-only monitoring credential, the flag-management token, and any staging-access creds. NEVER echo, print, log, or commit any of these — build headers inline from the env vars.

Evaluate A then B; act on the FIRST that applies. Refactor (D) runs after a successful Ship and as the idle fallback.

A) SHIP — `gh pr list --base <PRODUCTION_BRANCH> --state open --json number,title,labels,headRefName`. If an open PR whose head is a `<RELEASE_PREFIX>*` branch carries the `release:approved` label:
   1. LIGHT DELTA CHECK. The release branch is FROZEN, so the Release-ask review is authoritative — do NOT re-baseline against the integration branch. Determine whether any STABILIZATION commits landed on the release branch since the review comment was posted. If yes → run `/security-review` + the engineering review on JUST that delta. If none → the prior review stands.
   2. FLAG PLAN from the frozen branch: `git log origin/<PRODUCTION_BRANCH>..<release-branch> --oneline` mapped to `Closes #N`; the plan = each shipped feature's flag that is currently OFF in production.
   3. GATE. A release-blocking finding → do NOT merge. Remove the label (`gh pr edit <N> --remove-label release:approved`), comment the findings, open `ready-for-agent` bugfix issues, PushNotification "🚨 RELEASE HELD #<N> — <k> blocking finding(s); re-approve after triage", STOP. Fix only trivial, clearly-safe findings ON THE RELEASE BRANCH (within `config.escalation.maxFixCycles`) and re-check.
   4. MERGE. Clean → `gh pr merge <N> --merge` (a MERGE COMMIT, not squash — squashing into production makes it stop being an ancestor of the integration branch). Merging to production auto-deploys via CI — confirm the prod deploy via <DEPLOY_CHECK>. Migrations run via CI, not from this machine.
   5. FLIP + SMOKE-TEST. Execute the flag plan YOURSELF: <FLAG_FLIP_NOTES>. Never touch platform-maintenance flags. Smoke-test prod observability-only (no prod login): confirm the shipped features are ON and new endpoints respond (401/404 not 5xx). Close-watch the prod error rate via the read-only monitoring credential for ~`config.monitoring.closeWatchMinutes` (poll every ~`config.monitoring.pollSeconds`). Open `ready-for-agent` bugfix issues for non-critical findings; contain a critical one by flipping its feature flag OFF first.
   6. BACK-MERGE production → integration (Gitflow). So the integration branch receives any release-branch-only stabilization commits and production stays an ancestor of it. Create `chore/sync-<PRODUCTION_BRANCH>-into-<INTEGRATION_BRANCH>-$(date +%F)` off `origin/<PRODUCTION_BRANCH>`, push it, open a PR `--base <INTEGRATION_BRANCH>`, and merge with a MERGE COMMIT. If it conflicts, resolve in an ephemeral worktree. The shipped issues close automatically (`Closes #N` reached production).
   7. Delete the merged release branch: `git push origin --delete <release-branch>`.
   8. POST-RELEASE REFACTOR. Now run ONE conservative cleanup pass — go to D (post-release is the canonical refactor moment).
   9. PushNotification the outcome ("✅ SHIPPED #<N> (+<k> more) — prod healthy" or "🚨 RELEASE PROBLEM: …"). STOP.

B) RELEASE-ASK — if there is NO open `<RELEASE_PREFIX>*`→`<PRODUCTION_BRANCH>` PR, and `git log origin/<PRODUCTION_BRANCH>..origin/<INTEGRATION_BRANCH> --oneline` is non-empty:
   1. STAGING HEALTH. Verify the latest integration deploy is green via <STAGING_HEALTH>, plus a read-only error-rate check. Unhealthy → PushNotification "🚨 STAGING UNHEALTHY: …" and STOP.
   2. CUT THE RELEASE BRANCH (freeze). `VER=<RELEASE_PREFIX>$(date +%F)` (append `-2`, `-3` if that branch already exists today). Create it off the CURRENT integration tip and push, without touching the live working tree: `git branch "$VER" origin/<INTEGRATION_BRANCH> && git push origin "$VER" && git branch -D "$VER"` (the pushed remote branch backs the PR; the local ref is deleted so the live branch list stays clean). This FREEZES the shipping set — later Build merges won't change it.
   3. OPEN THE RELEASE PR. `gh pr create --base <PRODUCTION_BRANCH> --head "$VER"`. Body: included commits mapped to `Closes #N`, and a `## Flag plan` section listing each flag going OFF→ON in production at ship.
   4. REVIEW (AUTHORITATIVE — the branch is frozen). Over `git diff origin/<PRODUCTION_BRANCH>...$VER` (read-only): run `/security-review` AND a general engineering PR review (spawn a subagent per `config.models.subagents`: regressions via caller-tracing, logic/correctness bugs, unhandled edge cases & error paths, breaking API/schema/migration/contract changes, perf traps like N+1 / unbounded queries on hot paths, missing test coverage; flag anything against `CLAUDE.md`/`CONTEXT.md`). Post findings as a PR comment grouped 🔒 Security / 🛠️ Engineering. Fix only trivial, clearly-safe findings ON THE RELEASE BRANCH within `config.escalation.maxFixCycles`; leave the rest as comments. If anything is release-blocking, say so in the comment and the Push. Note in the PR body that this review is authoritative because the branch is frozen, and any later stabilization commits get a delta re-check at Ship.
   5. STAGING EVIDENCE (only if `<STAGING_URL>` is set). Because flags ship on-in-staging, the integration deploy runs every feature. For each shipped `Closes #N`, capture proof on `<STAGING_URL>` (pass any staging-access headers inline from env, never hard-coded; log in with the dev fixture if a non-production dev login is enabled — never solve a CAPTCHA/OAuth/magic link). Capture into `EVID=$(mktemp -d)`, DISPLAY the key screenshots inline at the end of the run (Read the PNG files so they render in this run's transcript — never committed, never hosted), add a PR note pointing to the run, then `rm -rf "$EVID"`. If staging is unreachable, comment "⚠️ staging evidence skipped — …" and continue.
   6. Do NOT merge. PushNotification (plain text): "✅ RELEASE PR #<N> READY — staging evidence in the run + security & engineering review on the PR — add `release:approved` to ship" (prepend "⚠️ <N> blocking findings — " if the review flagged any). STOP.

D) REFACTOR — reached after a successful Ship (A.8) OR when neither A nor B applied (idle). One conservative, behavior-preserving cleanup pass, rate-limited so it never churns:
   - GUARD — skip (idle, silent) if ANY holds: an open refactor PR exists (head `feature/refactor-`); a `feature/refactor-*` PR was merged in the last ~24h (at most one refactor/day); or the integration branch has no new commits since that last refactor merged.
   - Otherwise run ONE pass: `git worktree add -b feature/refactor-$(date +%Y%m%d) .claude/worktrees/refactor-$(date +%Y%m%d) origin/<INTEGRATION_BRANCH>`, symlink dependencies + any dev secret, `cd` in and run `/refactor-pass` — conservative, behavior-preserving, no feature changes. Keep `<TEST_CMD>` + `<BUILD_CMD>` green throughout (that IS the safety gate). Dead flags (fully released + stable) deleted code-first via the flag API. Push unsafe targets to `icebox` issues.
   - If it produced changes: push, open a PR into `<INTEGRATION_BRANCH>` (title `refactor: <summary>`, no `Closes`), run `/security-review` on the diff, then `gh pr merge <pr> --squash --auto`. Found nothing worth doing: open no PR.
   - CLEANUP (always): `git worktree remove .claude/worktrees/refactor-* --force` + `git branch -D feature/refactor-*`. Silent — unless the pass can't keep tests green: abandon the branch, run CLEANUP, PushNotification "🚨 refactor pass couldn't stay green — skipped".

GUARDRAILS:
- This loop runs on the PRIMARY machine ONLY — never on a secondary build machine (avoids double-ship / double-flag-flip races).
- Never deploy or migrate from this machine — the local credential is READ-ONLY; production deploys happen only via CI on merge to production. `/release` only merges the PR and watches CI/check-runs.
- The one mutation against the platform is flipping feature flags (owner standing order — never AskUserQuestion). NEVER touch platform-maintenance flags — maintenance stays ON until the owner launches.
- integration↔production movement uses MERGE COMMITS (`<RELEASE_PREFIX>*`→production and the production→integration back-merge); feature/refactor→integration use `--squash`.
- Production is observability-only: never act as a user in production, never touch real-user data.
- Treat env values as secrets: source/symlink them, never echo/print/log/commit them.
- Always remove any worktree you created AND delete its local branch ref before finishing. Commit messages end with the project's required Co-Authored-By trailer.
