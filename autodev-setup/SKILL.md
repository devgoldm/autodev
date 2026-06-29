---
name: autodev-setup
description: Set up a project (new or existing) for autonomous vision-driven development — VISION.md, CONTEXT.md/ADRs/PRDs, main/develop/feature branching with feature flags, and project commands (/vision, /build-next, /release, /bug-hunt, /refactor-pass) that run build/verify loops via subagents. Stack-, git-host-, and runtime-neutral via swappable presets (Cloudflare + GitHub + Claude Code shipped as the reference presets). Use when the user wants to "set up autodev", make a project run autonomously from a vision, or scaffold the vision-driven workflow on a new or existing codebase.
---

# Autodev Setup

Sets a project up so the owner only writes and refines a **VISION.md** while agent sessions propose features, build them behind flags, release them, monitor production, and keep the codebase clean. The operating model lives in [METHODOLOGY.md](METHODOLOGY.md) — read it before doing anything, and copy it into the project so the project commands can read it too.

## How this skill is organized

The methodology and commands are **neutral**. Everything specific to a particular platform, git host, or automation runtime lives in a **preset** under [presets/](presets/), selected during the interview and recorded in `config.json`:

- **Stack preset** — what the app is built and deployed on. [presets/stack-cloudflare.md](presets/stack-cloudflare.md) is the worked reference (Cloudflare Workers + Hono + Vite/React + flags + preview isolation). [presets/stack-generic.md](presets/stack-generic.md) is a fill-in template for any other stack, and the path for every existing project (detect and record, don't impose).
- **Git-host adapter** — where the repo, work tracker, and PRs live. [presets/githost-github.md](presets/githost-github.md) is the reference (GitHub via `gh`). The concepts (issues, labels, PRs, the atomic claim protocol) are host-neutral; the adapter supplies the commands.
- **Loop runtime** — what executes the three autonomous loops. [presets/runtime-claude-code.md](presets/runtime-claude-code.md) is the reference (Claude Code scheduled tasks + Push + label approval). [presets/runtime-cron-ci.md](presets/runtime-cron-ci.md) covers running the same loops under plain cron or CI.

When a step below says "per the stack preset" / "per the git-host adapter" / "per the loop runtime", read the selected preset for the concrete commands. If the project's choice has no shipped preset, write a short one from the generic template and save it next to the others.

## Process

### 1. Detect & interview

Determine if this is a new (empty dir / no app code) or existing project. Then interview the owner with AskUserQuestion. Ask, at minimum:

- **Technical level**: technical / semi-technical / non-technical. Calibrates all future grilling and explanations. Stored in config.
- **New projects**: what is the app? (1-2 sentences seeds VISION.md), name/domain, auth needed?
- **Stack**: which stack preset applies. Offer the Cloudflare reference preset as one option, "my own stack" (generic template) as another. For existing projects, **detect** the stack and **respect it absolutely** — never migrate or restructure unless the owner asks or the task is literally impossible on it. Record in `config.stack`.
- **Git host**: where the repo + work tracker live (GitHub is the reference; record the adapter in `config.gitHost`). If the project has no remote yet, offer to create one on the chosen host.
- **Loop runtime**: how the autonomous loops should run — your agent's own scheduled-automation feature (Claude Code scheduled tasks/routines, Cursor automations, Codex, …; the worked reference is Claude Code) or plain cron/CI. Record in `config.loop.runtime`. The interactive commands work regardless; the runtime only governs the unattended cascade.
- **Prod test access**: dedicated seeded test account (default) or a network-gate access token, per [SETUP-REFERENCE.md](SETUP-REFERENCE.md).
- **Work tracker**: where the work queue lives. **Default: the git host's issues**, which is event-native and what existing repos usually already use. Only fall back to a `BACKLOG.md` file if the repo has no issue tracker. Record in `config.workTracker`. See § Work tracking in the ORCHESTRATION template.
- **Build-loop count & frequency** (if the loop runtime is machine-local, like Claude Code scheduled tasks): the cascade is **three loops** — the **build** loop, a daily **release** loop, and a daily **bug-hunt** loop. Ask two things about the build loop on this machine: **how many** should run in parallel and **how often** each runs. Default **one loop, hourly**. If they ask for more, register that many `autodev-build` tasks, each with a distinct **worker suffix** + **dev port**, and **auto-stagger** them evenly across the interval (e.g. two loops every 30 min → offset 15 min apart). Record both in `config.loop.build` (`instancesPerMachine`, `everyMinutes`). They can add more machines later (see step 7). The **release** loop runs on the **primary machine only** (it owns prod monitoring + the approval gate). Confirm which machine is primary, and whether they want a persistent **named front-door session** on it for the live grill. See the loop runtime preset for the specifics.

### 2. Check prerequisites

See [SETUP-REFERENCE.md](SETUP-REFERENCE.md) § Prerequisites for the universal list (a git-host CLI, a browser-automation tool, the context-budget setting). Autodev's grilling/refactor/QA capabilities are **bundled** (the `grill` skill + the commands themselves) — no external skills to install. Then check the **preset-specific** prerequisites in each selected preset (e.g. the stack preset's toolchain + credentials, the git-host adapter's CLI auth, the loop runtime's scheduler). Check and report; offer to install what's missing rather than failing.

### 3. Scaffold or onboard

- **New project**: scaffold per the **stack preset** § New project scaffold. Always include a test runner, a flags module wired to the chosen flag mechanism, and a seed script for the test account if there's auth.
- **Existing project**: full onboarding pass. Explore the codebase and reverse-engineer `CONTEXT.md`, seed `docs/adr/` with the evident load-bearing decisions, and **draft** VISION.md from what the app observably does — then walk the owner through it for correction. Detect the existing stack, flag mechanism, test setup, and deploy pipeline and record them in `config.json` (write a generic stack preset for them); do **not** rewrite their code. See [SETUP-REFERENCE.md](SETUP-REFERENCE.md) § Existing project onboarding.

### 4. Write the autodev files

Copy from this skill's `templates/` directory into the project:

| Source | Destination |
|---|---|
| `templates/VISION.md` | `VISION.md` (root; or merge with onboarding draft) |
| `templates/BACKLOG.md` | `BACKLOG.md` (root) — **only if** the repo has no issue tracker; the default work queue is the git host's issues. Record the choice in `config.workTracker`. |
| `templates/PRD-FORMAT.md` | `.claude/autodev/PRD-FORMAT.md` |
| `METHODOLOGY.md` (this skill's) | `.claude/autodev/METHODOLOGY.md` |
| The selected **presets** (stack / git-host / runtime) | `.claude/autodev/presets/` (so the project's commands can read them) |
| `templates/config.json` | `.claude/autodev/config.json` (fill in interview answers + preset choices) |
| `templates/ORCHESTRATION.md` | `.claude/autodev/ORCHESTRATION.md` (fill in repo path, branch names, staging URL, front-door session name) |
| `templates/grill/` | `.claude/skills/grill/` (autodev's bundled grilling skill, used by `/vision`) |
| `templates/vision/` | `.claude/skills/vision/` |
| `templates/build-next/` | `.claude/skills/build-next/` |
| `templates/release/` | `.claude/skills/release/` |
| `templates/bug-hunt/` | `.claude/skills/bug-hunt/` |
| `templates/refactor-pass/` | `.claude/skills/refactor-pass/` |
| `templates/scheduled-tasks/*` | the loop runtime's task location (e.g. `~/.claude/scheduled-tasks/` for Claude Code — see the runtime preset; fill in `<PLACEHOLDER>`s per step 6) |

Also copy any preset-specific project artifacts the selected presets call for (e.g. the Cloudflare preset's `cloudflare-token.md`). Create `docs/prds/` and `docs/adr/`. Templates are copied verbatim except `config.json`, `VISION.md`, `BACKLOG.md`, `ORCHESTRATION.md`, the preset artifacts, and the `scheduled-tasks/` SKILLs, which you fill in. The `scheduled-tasks/` templates are the operative prompts the loops run — place them where the loop runtime expects (the runtime preset says where).

### 5. Git & deploy plumbing

Per [SETUP-REFERENCE.md](SETUP-REFERENCE.md) § Git & deploys: init repo if needed, create `develop` from `main`, push both, set branch protection (PRs required into both), and connect the deploy pipeline (main → production, develop/PRs → previews) **per the stack preset**. Add `.env` to `.gitignore` and put test-account credentials there. Complete any preset-specific plumbing (e.g. the Cloudflare preset's least-privilege API token + CI secrets) and record it in `config.json`.

### 6. The autonomous loop & approvals

Set up the cascade so the owner's front door is a **live grilling conversation** and everything after the push runs autonomously as the **loop runtime**, contacting them **only at release**. Copy `templates/ORCHESTRATION.md` → `.claude/autodev/ORCHESTRATION.md` and fill in the repo path, branch names, staging URL, and front-door session name. See § The autonomous loop and § Notifications & approval in [SETUP-REFERENCE.md](SETUP-REFERENCE.md), and the **loop runtime preset** for how to register/schedule the three tasks.

- **Create the loops** from `templates/scheduled-tasks/` via the loop runtime's scheduler: one or more `autodev-build` (hourly) — default one on the primary, plus any extra build loops the owner asked for (more machines and/or several per machine, each a separate task with a distinct `<WORKER_SUFFIX>` + `<DEV_PORT>` on a **staggered** schedule); `autodev-release` (daily) on the **primary only**; `autodev-bug-hunt` (daily) on the **primary**. Replace every `<PLACEHOLDER>` with this project's values (repo path, branch names, release-branch prefix, staging URL + CI/check-run names, flag API + creds, seed command, dev-login fixture, dev ports, verify commands). Record them in `config.loop` (count in `config.loop.build.instancesPerMachine`).
- **Notifications & approval** are runtime-specific — see the runtime preset (e.g. Claude Code = built-in Push + a `release:approved` label on the `release/*`→`main` PR). Create the approval + lifecycle labels now via the git-host adapter. The trust boundary is repo write access — keep collaborators limited to people trusted to ship.
- **Front-door session:** optionally set up one persistent named session on the host machine as the grilling front door; otherwise the grill runs in any local session.
- **Staging auth for evidence (optional but recommended):** enable the dev test-account login on staging, env-gated to `development OR staging`, **never production** (see § Production access). Without it, staging evidence is public-surface + health only.

### 7. (Optional) Add more build loops

The build loop is **horizontally scalable** — add more loops to build more in parallel. Two ways, freely mixed, now or any time later:

- **More loops on this machine** — a **config knob**, not a copy-paste. Bump `config.loop.build.instancesPerMachine` (and/or lower `everyMinutes`) and register that many `autodev-build` tasks, each with a distinct `<WORKER_SUFFIX>` (`-2`, `-3`, …) and `<DEV_PORT>`, **auto-staggered** evenly across `everyMinutes`. This is the same thing the setup interview asked; it's just re-run to change the numbers.
- **Another machine** (e.g. a teammate's, on their own token budget) — they need access to the repo. They **check it out**, then paste the **build-worker prompt** (in the README's *Get started*), which installs the skill if needed, registers **only** an `autodev-build` loop there, installs dependencies, and sets up the gitignored env that loop needs — the flag-management token (if the project manages flags via an API) plus whatever local secrets the dev server needs to run. *Not* the monitoring or staging-access creds — those belong to the primary's release/bug-hunt loops, not a builder.

In all cases register **only** `autodev-build` on the extra loop — never `autodev-release` or `autodev-bug-hunt`, which stay on the **primary** (it owns prod monitoring + the approval gate). The **atomic claim, keyed on the per-loop worker id** (git-host adapter § Claiming work), guarantees no two loops ever build the same ticket — no global lock, no central coordinator. Removing a loop is just deleting its `autodev-build` task (in-flight claims auto-reclaim after the TTL).

### 8. Hand over

Commit everything as the setup commit (on `develop`, PR'd to `main`). Then tell the owner, at their technical level, how the workflow runs:

1. Whenever an idea or bug strikes, they **talk to the front-door session** ("I want feature X", "bug Y appeared") and get **grilled there and then**; it updates `VISION.md` if warranted, writes the PRD, opens a `ready-for-agent` work item, and pushes.
2. The hourly **build loop** (`autodev-build`, on every build machine) atomically claims the top `ready-for-agent` item, builds it in an ephemeral worktree (flag on-in-staging/off-in-prod), verifies locally, and merges to the integration branch on green CI → staging deploy. Extra machines just build more in parallel.
3. The daily **release loop** (`autodev-release`, primary only): when the integration branch is ahead, it cuts a **frozen `release/<version>` branch**, opens the `release/*`→`main` PR with a security review + authenticated staging evidence (shown in the run), and **notifies once**. They approve (per the runtime's approval mechanism) to ship; the next release run merges to `main`, flips prod flags, close-watches prod, back-merges to `develop`, and runs a refactor pass.
4. The daily **bug-hunt loop** (`autodev-bug-hunt`, primary) keeps prod healthy (logs-only → `ready-for-agent` bug items), and `/refactor-pass` keeps the code legible — run by the release loop after each ship and when it's otherwise idle.
5. `VISION.md` stays the living north-star but is maintained by the grill, not edited to trigger work.
6. If the loop runtime is machine-local, the loops only tick while it's running on each host — keep it open (they resume on next launch).

## Review checklist

Before finishing: config.json filled in (including the selected presets), all five project commands present, VISION.md has real content (not placeholder), CONTEXT.md exists, develop branch exists and is default for PRs, the flags mechanism actually works (one `example` flag wired end-to-end — unless `flags.mechanism: "none"` was recorded), the test account can log in, and any **preset-specific** setup is complete and recorded (e.g. Cloudflare least-privilege API token minted, placed in CI secrets + local, verified, and documented). Also: `ORCHESTRATION.md` filled in; the approval + lifecycle labels created; and the three loops registered with the loop runtime from the filled-in templates and recorded in `config.loop` (with `loop.claim` + `loop.release.host`).
