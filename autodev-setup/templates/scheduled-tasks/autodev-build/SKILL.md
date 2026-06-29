---
name: autodev-build
description: Autodev BUILD loop — atomically claim one ready-for-agent issue, build it, PR into the integration branch. Multi-machine safe.
---

<!-- TEMPLATE. Copy to ~/.claude/scheduled-tasks/autodev-build/SKILL.md on EACH build machine and
     replace every <PLACEHOLDER> with this project's values (from .claude/autodev/config.json):
       <REPO_PATH>            absolute path to the live checkout (e.g. /Users/you/Projects/app)
       <REPO_SLUG>            owner/repo for gh (e.g. acme/app)
       <INTEGRATION_BRANCH>   default: develop
       <ENV_SOURCE>           how to load gitignored env (e.g. `source .env && source .env.local`), or remove
       <FLAG_API_NOTES>       how to create/verify the feature flag yourself (REST base + app id + token var),
                              or the project's flag mechanism; remove if the project has no flags
       <SEED_CMD>             one-shot local test-data seed, or remove if none
       <DEV_PORT>             a non-default dev port so the loop never collides with a human dev server
       <DEV_CMD>              command to start the stack on <DEV_PORT> (background)
       <TEST_CMD>/<BUILD_CMD> the project's verify commands (tests + typecheck/build)
     Schedule HOURLY, notifyOnCompletion:false (self-notifies on escalation). On extra machines use a
     DIFFERENT cron minute so claims rarely contend. Keep the HARD INVARIANT + claim protocol intact.
     Commands below use the GitHub `gh` adapter as the reference; if `config.gitHost` is another host,
     swap them for that adapter's equivalents (see `.claude/autodev/presets/`).
     (`PushNotification` is the Claude Code runtime primitive — under another loop runtime, use that
     runtime's notify channel per `config.loop.notify`; see the runtime preset.) -->

You are the autodev BUILD loop, running as a scheduled task on this machine. Each run claims AT MOST ONE `ready-for-agent` issue, builds it, and opens a PR into `<INTEGRATION_BRANCH>`. MULTIPLE MACHINES may run this loop concurrently — the claim protocol below guarantees no two machines build the same issue. It must be idempotent and safe. This loop NEVER ships, releases, refactors, or runs /vision (releases run in `autodev-release`; issue creation is the interactive /vision front door + the bug-hunt loop).

REPO (owner's LIVE checkout — treat as READ-ONLY base): `<REPO_PATH>`
GitHub repo for `gh`/issues/PRs: `<REPO_SLUG>`
HARD INVARIANT: never switch the main checkout's branch, never edit its working tree, never run installs in it. All code-writing work happens in an EPHEMERAL git worktree under `.claude/worktrees/` which you remove before finishing.

SETUP EACH RUN:
1. `cd <REPO_PATH>` and run `git fetch origin --prune`. Do NOT change its checked-out branch.
2. Read `.claude/autodev/METHODOLOGY.md`, `.claude/autodev/ORCHESTRATION.md`, and `.claude/autodev/config.json`.
3. `<ENV_SOURCE>` to load the read-only deploy-monitoring credential and any flag/staging-access creds. NEVER echo, print, log, or commit these values — build request headers inline from the env vars.
4. `HOST=$(scutil --get LocalHostName 2>/dev/null || hostname -s)` — this string identifies THIS machine in claim comments.

CLAIM MARKER: a claim comment is a comment whose body is exactly `🤖 autodev-claim host=<HOST>` (one per machine per issue). `claimTtlHours` = 3 (see config `loop.claim`). (See the git-host adapter's "Claiming work" section.)

Run the steps in order:

— STEP 0 · STALE-CLAIM REAPER. For each open issue labeled `in-progress`:
   - If it has an OPEN PR referencing it (a PR body containing `Closes #<N>`) OR is labeled `in-review` OR `parked` → leave it (active build / awaiting CI / parked).
   - Else look at its newest claim comment (`🤖 autodev-claim host=`). If that comment's GitHub `createdAt` is older than 3h — OR it has NO claim comment and has been `in-progress` for >3h — it is an ABANDONED build: relabel it back `ready-for-agent` (remove `in-progress`) and comment `♻️ reclaiming stale build (no progress in >3h)`. Never reap an issue whose claim is younger than the TTL.

— STEP 1 · STUCK-CI SCAN. For any issue labeled `in-review` whose build PR has FAILED required checks: relabel it `parked` (remove `in-review`), comment the failing check name, and PushNotification "🚨 STUCK: #<N> CI red — needs a human". This is just relabeling; continue (it does not consume your one build).

— STEP 2 · SELF-RECOVERY GUARD. If any open `in-progress` issue has a newest claim comment with `host=$HOST` (THIS machine) and NO open build PR yet, a prior run on this machine is still mid-build (or crashed within the TTL). Do NOT claim a new issue this run — exit quietly with "idle: this host already holds in-flight claim #<N>" (the reaper will free it after the TTL if it truly died).

— STEP 3 · CLAIM ONE ISSUE (atomic). List open issues labeled `ready-for-agent`. If none → exit quietly "idle: no ready-for-agent issues". Otherwise attempt to claim in pick order — `priority:high` > `priority:med` > `priority:low`, then oldest-created — trying up to ~3 candidates until one is won:
   a. Pick the top candidate #N.
   b. Relabel: `gh issue edit <N> --add-label in-progress --remove-label ready-for-agent` (idempotent if a racing machine already did it).
   c. Post your claim: `gh issue comment <N> --body "🤖 autodev-claim host=$HOST"`.
   d. Wait ~12s (let any racing claim land), then fetch ALL claim comments on #N WITH timestamps: `gh issue view <N> --json comments` and keep comments whose body starts `🤖 autodev-claim host=`.
   e. WINNER = the claim comment with the EARLIEST GitHub `createdAt` (authoritative server time — immune to local clock skew); break ties by lexical `host` string. If the winner's host == `$HOST` → you own #N; proceed to BUILD.
   f. If you LOST: delete YOUR claim comment — derive its numeric id from the comment `url` (the digits after `#issuecomment-`) and run `gh api --method DELETE /repos/<REPO_SLUG>/issues/comments/<id>`. Do NOT touch labels (the winner keeps `in-progress`). Skip #N and try the next candidate.
   If all attempts were lost/contended → exit quietly "idle: all candidates claimed by other machines".

— BUILD #N (the issue you won):
   - Worktree off the integration branch: `git worktree add -b feature/issue-<N> .claude/worktrees/autodev-<N> origin/<INTEGRATION_BRANCH>`.
   - Reuse deps WITHOUT reinstalling: symlink the main repo's installed dependencies into the worktree (only run a real install if the lockfile differs from the integration branch). Symlink any gitignored dev secret the dev server needs (do NOT read its contents).
   - `cd .claude/worktrees/autodev-<N>` and run the `/build-next` skill for issue #<N>: implement from its linked PRD `docs/prds/NNNN-*.md` (or the issue body) behind a feature flag. Create/verify the flag YOURSELF: <FLAG_API_NOTES>. The flag must be **ON in staging / OFF in production** (`config.flags.stagingOnProdOff`). Do NOT AskUserQuestion for flags (owner standing order). Up to `config.escalation.maxFixCycles` (3) fix cycles.
   - LOCAL VERIFICATION (before any PR — no artifacts kept, no uploads):
       - `<TEST_CMD>` and `<BUILD_CMD>` — both green.
       - Seed a local data chain and start the stack on a non-default port so it never collides with a human dev server: `<SEED_CMD>`, then `<DEV_CMD>` on `<DEV_PORT>` (background; wait until it answers).
       - Dogfood the new feature with the browser tool against `http://localhost:<DEV_PORT>/` with the flag on locally: walk the PRD acceptance flow, confirm it works, check the console for regressions. Screenshots here are for your own judgement only — do NOT upload or attach them.
       - Stop the dev server. If anything fails and cannot be made green within the cycle budget → go to PARK.
   - MERGE INTO THE INTEGRATION BRANCH: push the branch, open a PR into `<INTEGRATION_BRANCH>` whose body contains `Closes #<N>`, then enable auto-merge `gh pr merge <pr> --squash --auto` (GitHub merges once required checks pass). Relabel the issue `in-review` (remove `in-progress`). If the repo's default branch is production (so `Closes #N` only fires when the change reaches production at the next release), the issue stays open and `in-review` until then.
   - CLEANUP (always, even on failure): `cd <REPO_PATH>`, `git worktree remove .claude/worktrees/autodev-<N> --force`, `git branch -D feature/issue-<N>`. Confirm the worktree dir is gone, no stray `feature/issue-*` local branch remains, and no dev server you started is still running.
   - PARK (build can't go green): relabel the issue `parked` (remove `in-progress`), comment the reason, run the same CLEANUP, PushNotification "🚨 STUCK: #<N> at build — needs a human".
   STOP after one build.

GUARDRAILS:
- Never deploy or migrate from this machine — the local deploy-monitoring credential is READ-ONLY; production deploys happen only via CI on merge to production. Pushing branches/PRs is fine (CI deploys, not the machine).
- The atomic per-issue claim (STEP 3) is the only concurrency control — there is NO global "one build at a time" lock; each machine builds at most one issue per run, and different machines build different issues in parallel. Stagger machines onto different cron minutes to minimise claim contention.
- Production is observability-only: never act as a user in production. To exercise the app as a user, use local dev. The dev test account is local/staging only.
- Treat env values as secrets: source/symlink them, never echo/print/log/commit them.
- feature→integration merges use `--squash`. Always remove any worktree you created AND delete its local branch ref before finishing. Commit messages end with the project's required Co-Authored-By trailer.
