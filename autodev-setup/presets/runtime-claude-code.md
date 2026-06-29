# Loop runtime — Claude Code scheduled tasks

The reference **loop runtime**: the three autonomous loops run as **Claude Code scheduled tasks** (the `mcp__scheduled-tasks` MCP), stored under `~/.claude/scheduled-tasks/` on the host machine, with built-in **Push** notifications and a **`release:approved` label** as the approval signal. No cloud routines, no chat infrastructure.

This is the worked reference for a whole **family** of runtimes: *your agent's own scheduled-automation feature.* The three task prompts in `templates/scheduled-tasks/` are just agent prompts on a schedule — nothing about them is Claude-specific. The same three drop into **any agent that can run itself on a timer**: Claude Code scheduled tasks / routines, **Cursor automations**, Codex's scheduled runs, or whatever ships next. Only three things differ per runtime and are abstracted into `config.loop`: how you **register/schedule** a task, the **notification** channel (`config.loop.notify`), and the **approval** signal (`config.loop.approval`). Pick the agent you actually use; if it isn't Claude Code, copy this file to `runtime-<agent>.md`, swap those three, and keep the rest. (For a headless server with no agent automation, use [runtime-cron-ci.md](runtime-cron-ci.md).)

Record `config.loop.runtime` = this file (or your `runtime-<agent>.md`), `config.loop.notify` = `"push"` (or your channel), `config.loop.approval` = `"label:release:approved"`.

## Why local, not cloud

Cloud routines were tried and abandoned: the remote build environment kept failing because a production install omits `devDependencies` (the whole toolchain + lifecycle scripts). Running locally sidesteps that — the machine already has the repo, dependencies, the git-host CLI, the browser tool, and the read-only monitoring credential. The tradeoff: the loops only tick **while the Claude app is open** on that machine (they resume on next launch). For a working-machine setup that's the right trade. (To run unattended on a server instead, see [runtime-cron-ci.md](runtime-cron-ci.md).)

## Registering the tasks

Create all three via `mcp__scheduled-tasks`, copying each filled-in template from `templates/scheduled-tasks/` to `~/.claude/scheduled-tasks/<name>/SKILL.md`:

- **`autodev-build`** — `30 * * * *` (hourly), `notifyOnCompletion:false` (self-notifies on escalation). Run one per build loop, each at a **different cron minute** so claims rarely contend. Default is one loop per machine; to add more throughput, create extra `autodev-build` tasks — on more machines, and/or **several on one machine**, each with a distinct `<WORKER_SUFFIX>` (`-2`, `-3`, …) and its own `<DEV_PORT>`. Coordination is by worker id, so same-machine loops never collide.
- **`autodev-release`** — `45 9 * * *` (daily, machine-local — cron follows the clock through DST), `notifyOnCompletion:false`, **primary machine only** (it owns prod monitoring + the approval gate).
- **`autodev-bug-hunt`** — daily, `notifyOnCompletion:true`, **primary**.

The loops only ever run on machines where the owner keeps the Claude app open. Secondary machines run **only** `autodev-build`.

## Notifications — Push

The `autodev-build` and `autodev-release` tasks are **silent on idle** and send a `PushNotification` only when a release PR is ready to approve or a build escalates; `autodev-bug-hunt` Pushes once per run. Push is plain text (it points at the PR / run; it **cannot carry images** — staging screenshots are shown in the release run's transcript instead).

## Approval — the `release:approved` label

The owner adds the **`release:approved` label** to the `release/*`→`main` PR (one tap on GitHub mobile/web). The next `autodev-release` run sees it and ships. **The trust boundary is repo write access** — anyone who can label the PR can ship, so keep collaborators limited to people trusted to release. (No per-user gating, webhook, or bot.) Create the label during setup (see the git-host adapter).

## Front-door session

The owner's **front door (grilling) is NOT a scheduled task** — it's a live, owner-initiated interactive session so `AskUserQuestion` works. Optionally run one persistent **named session** on the host as the front door (reachable from the phone app); otherwise the grill runs in any local session. Record `config.loop.frontDoor`.

## Host facts the templates use

- **Worker id** (claim identity): `$(scutil --get LocalHostName 2>/dev/null || hostname -s)<WORKER_SUFFIX>` — the hostname (macOS/Linux) plus an optional suffix that's empty for a machine's default build loop and `-2`/`-3`/… for additional loops on that same machine.
- **Worktrees** under `.claude/worktrees/` in the live checkout; removed (plus the local branch ref) before each run finishes.
- **Read-only monitoring credential** + flag-management token + any staging-access token live in the host's **gitignored** env, loaded via `<ENV_SOURCE>` (e.g. `source .env && source .env.local`). Never echoed, printed, logged, or committed.
