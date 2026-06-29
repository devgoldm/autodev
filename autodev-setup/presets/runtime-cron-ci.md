# Loop runtime — cron / CI

An alternative **loop runtime** for running the three loops **unattended on a server** (or in CI) instead of in an open Claude app. Same three tasks, same scheduled-task prompts (`templates/scheduled-tasks/`), same per-issue claim protocol — only the scheduler, the notification channel, and the approval signal differ. Use this when no one will keep an interactive app open, or when you want the loop on always-on infrastructure.

Record `config.loop.runtime` = this file (or a `runtime-<name>.md` you derive from it).

## Running an agent from cron/CI

The loops are agent prompts, so the runtime must invoke an agent non-interactively on a schedule:

- **cron + headless agent CLI** on a server: a crontab entry per build loop runs the agent against the task prompt (e.g. `0 * * * * cd <repo> && <agent-cli> --skill autodev-build`). Run as many build loops as you want — across servers and/or several on one host (each with a distinct `<WORKER_SUFFIX>` + `<DEV_PORT>`) — staggered onto different minutes exactly as in the machine-local runtime.
- **CI scheduled workflows**: a scheduled pipeline (GitHub Actions `schedule:`, GitLab scheduled pipelines, etc.) runs the agent in a job. `autodev-build` can fan out across runners; `autodev-release` + `autodev-bug-hunt` run as single scheduled jobs.

The build loop's worktree isolation matters less on disposable CI runners (each job is a fresh checkout) but keep it for server-cron, where the host has a live checkout.

## The devDependencies trap (read this)

This is the failure mode that pushed the reference design to machine-local. A production-mode install (`npm ci --omit=dev`, `pip install` without dev extras, etc.) **omits the toolchain** — typechecker, test runner, linter, bundler — and lifecycle scripts (`postinstall`, `prepare`/husky). The loops need all of it. In any cron/CI runtime, **force dev dependencies** and disable hook-skipping in the install step:

```bash
HUSKY=0 npm ci --include=dev      # node example; adapt per stack
```

Verify the verify-commands (`<TEST_CMD>`, `<BUILD_CMD>`) actually run in the runtime before trusting the loop.

## Notifications

No built-in Push here — pick a channel the runtime can post to and wire it where the templates say "notify":

- A chat webhook (Slack/Discord/Teams), email, or an SMS/push service.
- A CI annotation / job summary (visible, but not a phone alert).

Set `config.loop.notify` to the channel. Keep the discipline: **silent on idle**, signal only on release-ready / escalation / once-per-bug-hunt. Like Push, most channels are text-only — keep staging evidence in the run log/artifacts, not the message.

## Approval

No label-tap-from-an-app assumption. Options, in rough order of simplicity:

- **A label on the release PR** (`release:approved`), same as the reference — works on any host with labels; the next scheduled `autodev-release` run reads it. This is the recommended default.
- **A manual workflow-dispatch / pipeline trigger** the owner runs to authorize the ship.
- **A protected-environment approval** (e.g. a GitHub Actions environment requiring a reviewer) gating the ship job.

Set `config.loop.approval` accordingly. The trust boundary is still **whoever can produce that signal** — scope it to people trusted to release.

## What stays identical

Everything in the methodology and the scheduled-task templates: Gitflow + frozen release branch, the atomic per-issue claim, two-tier verification, the flag plan, observability-only production, the back-merge, the refactor pass. Only scheduler + notify + approval are swapped here.
