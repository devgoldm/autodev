# Autodev Methodology

The operating model for this project. Every autodev command (`/vision`, `/build-next`, `/release`, `/bug-hunt`, `/refactor-pass`) reads this file first. In a project, this file lives at `.claude/autodev/METHODOLOGY.md` alongside `config.json` (owner profile, chosen presets, escalation settings).

This file is **stack-, git-host-, and runtime-neutral**. Anything specific — what framework you build on, where issues and PRs live, what runs the autonomous loop — is supplied by a **preset** recorded in `config.json` and described under `presets/`. The methodology refers to those choices by role (the *stack*, the *git host*, the *loop runtime*, the *flag mechanism*, the *deploy pipeline*, the *observability source*), never by product name.

## Vocabulary

These roles are filled by a preset; the rest of this document only uses the role name.

| Role | What fills it | Preset field |
|---|---|---|
| **Stack** | The language/framework/platform the app is built and deployed on | `config.stack` (e.g. `presets/stack-cloudflare.md`, or `stack-generic.md` filled in) |
| **Git host** | Where the repo, work tracker, and PRs live | `config.gitHost` (e.g. `presets/githost-github.md`) |
| **Work tracker** | The queue of features/bugfixes (host issues, or a `BACKLOG.md` file) | `config.workTracker` |
| **Loop runtime** | What executes the autonomous scheduled loops | `config.loop.runtime` (e.g. `presets/runtime-claude-code.md`, `runtime-cron-ci.md`) |
| **Flag mechanism** | How feature flags are created/flipped/deleted | `config.flags.mechanism` |
| **Deploy pipeline** | What turns a merge into a deployment | `config.deploy.mechanism` |
| **Observability source** | Where production logs/errors are read from | `config.prodTest` / the stack preset |

## Roles (people & agents)

- **Owner** — writes and refines `VISION.md`. Approves feature designs and releases. That's the whole job. Communicate with them at the technical level recorded in `config.json` (`owner.technicalLevel`): in-depth and precise for technical owners, plain-language outcomes and trade-offs for non-technical ones. When you need them, use AskUserQuestion.
- **Orchestrator** — the interactive session running an autodev command. Runs on whatever model the owner chose; does the judgment work (proposing, grilling, reviewing, escalating) and spawns subagents for the labor.
- **Subagents** — do development, verification, and bug-fixing. Pinned to the model in `config.models.subagents` (default the most capable available model) so orchestrator tokens aren't burnt on labor.

## Token discipline

- **Serial, one branch.** Subagents run one at a time. In an **interactive** session they work on the current checkout and branch in place — no worktrees, no parallel file-mutating agents (serialization saves tokens-per-minute and avoids workspace-replication headaches). The **autonomous loop** is the one exception: because it runs on a machine someone may also be using, its build step works in an **ephemeral git worktree** off the integration branch (reusing installed dependencies, torn down after the PR) so it never touches the live checkout's branch or tree. That is isolation, not parallelism — **one build per loop per run**. The build loop is **horizontally scalable**: run as many build loops as you want — across extra machines and/or several on one machine (default: one per machine, which keeps token spend predictable) — each on a staggered schedule. An **atomic claim keyed on a per-loop worker id** (see the git-host adapter's *Claiming work* section) guarantees no two loops ever build the same issue; because it keys on the worker id rather than the machine, loops sharing a machine never collide (each needs a distinct worker suffix + dev port).
- Subagent prompts must be **self-contained**: PRD path, task text, relevant file paths, verify commands, and the rule that they work on the current branch in place.
- Don't re-derive context: PRDs, CONTEXT.md, and the work tracker exist so sessions can pick up cold.

## Context hygiene

Performance and cost degrade as a session's context grows, so keep windows small:

- **Subagents are the main lever**: all heavy labor (implementation, verification, big file reads) runs in subagent windows, so the orchestrator's own context grows slowly.
- **Auto-compact early**: if the harness supports it, projects set `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=20` in `.claude/settings.json`, so sessions compact at 20% of the context window rather than near the limit. State lives in the repo docs and the work tracker (PRD Build log, issues, CONTEXT.md), so a compaction loses little — keep those current as you work.
- If a compaction is coming at an awkward moment, suggest the owner run `/compact focus on <the in-flight feature>` first — a focused compaction beats an unguided one.

## The build/verify loop (fractal)

The same loop shape applies at every level:

1. **Build** — implement the change.
2. **Verify** — actually run it: tests, then the app itself. A feature is not done because the code looks right; it's done when it's been observed working.
3. **Fix** — if verification fails, fix and re-verify. Repeat.

Subagents run this loop on each atomic task. The orchestrator then runs the same loop on the whole feature — spawning fresh verify-subagents and fix-subagents. Always add automated tests alongside features whenever possible.

**Escalation (3 strikes):** if the orchestrator-level fix loop fails `escalation.maxFixCycles` (default 3) times on the same feature, stop burning tokens. Park the feature branch, record what was tried in the PRD's Build log, set the PRD status to `parked`, and AskUserQuestion the owner with options: simplify scope, try a different approach, or drop it.

**Two-tier verification (the autonomous loop).** Nothing reaches production un-exercised:

1. **Local — before the integration merge.** In the build worktree, start the stack, seed test data, and exercise the new feature with a browser-automation tool alongside the test suite + typecheck + lint. Only open the PR if green; otherwise park. **No artifacts kept** — this tier exists so nothing broken merges. The PR then auto-merges to the integration branch on green CI.
2. **Staging — before the release ask.** Because flags are on-in-staging, the staging deploy runs the feature. Drive staging with the browser-automation tool, logged in as the dev test account (enabled on staging per § Production access), and capture a short authenticated walkthrough + screenshots per shipped issue. **Show the evidence in the run transcript** — never commit it or host it externally (that bloats history). The release PR carries a security review + a pointer to that run, so the owner verifies before approving.

If the project has **no flag mechanism and no staging environment**, this collapses to a single tier (local verification before merge) plus post-release production monitoring — record that in `config.json` and skip the staging-evidence steps. The two-tier model is the default because it is safest, not because it is mandatory.

## Branching & flags

**Gitflow.**

```
feature/<slug>  →  develop  →  release/<version>  →  main (production)
                                      └──────── back-merge ───────→ develop
hotfix/<slug>   →  main  (+ back-merge → develop)
```

Branch names are configurable (`config.deploy.productionBranch` / `integrationBranch` / `releaseBranchPrefix`); this document uses the defaults `main` / `develop` / `release/`.

- All movement between branches is via **PRs**. Never push directly to develop or main.
- **The release is cut as a `release/<version>` branch off `develop`, which freezes the shipping set.** Build keeps merging `feature/*` → `develop` (squash) the whole time, but those merges do **not** touch the open release branch — so the release review is **authoritative** (no re-baseline against a moving `develop` at ship time) and the flag plan is derived once from the frozen branch. Any stabilization fixes found in review/staging are committed **on the release branch** so the reviewed set === the shipped set. Default version scheme: date-based `release/YYYY-MM-DD` (suffix `-2`/`-3` for multiple per day) while a product is pre-launch/unversioned; trivially swappable for semver.
- **Ship** merges `release/<version>` → `main` with a **merge commit** (never squash — squashing into `main` makes it stop being an ancestor of `develop`), then **back-merges `main` → `develop`** (a `chore/sync-main-into-develop-<date>` PR, merge commit) so develop picks up any release-branch-only fixes and `main` stays an ancestor of `develop`. **Hotfixes** for critical production bugs branch off `main`, PR back to `main`, and back-merge to `develop` the same way. Non-critical bugs become bugfix items in the work tracker.
- Every feature is gated behind a **feature flag** (mechanism in `config.flags`) so feature → develop merges can happen as fast as possible without exposing unfinished work. Flag-off must mean invisible-and-harmless in production.
- **Flags ship on-in-staging / off-in-production** (`config.flags.stagingOnProdOff`): a new flag is created on in the staging/preview environment and off in production, so the staging deploy actually exercises the feature (and the loop can capture end-to-end evidence on it) while production stays dark. The **release flips the production flag on** as part of its flag plan, so one approval ships the staging-proven feature live.
- **The agent manages flags itself** via the flag mechanism's API/SDK (credentials in the host's gitignored env) — create/flip/delete without AskUserQuestion (owner standing order). Some projects enforce staging-on/prod-off **in code** (an `APP_ENV !== "production"` predicate forces feature flags on outside prod), in which case a new flag only needs to *exist* off to be prod-dark while staging runs it on; record the project's mechanism in `config.flags`. If the project genuinely has no flag system, record `flags.mechanism: "none"` and rely on the frozen release branch + staging gate for safety instead — but adding even a trivial flag module is strongly recommended, because it is what makes fast feature→develop merges safe.
- **Review every PR — security AND general engineering**: run a security review plus a general engineering review (regressions, logic/correctness bugs, unhandled edge cases & error paths, breaking API/schema/migration changes, performance traps, missing test coverage — what a careful human PR reviewer checks). Fix trivial findings in the same PR; surface the rest for the owner. In the autonomous loop the release review runs on the **frozen `release/<version>` branch** (authoritative — it cannot grow under the reviewer), with only a light delta re-check at ship for any stabilization commits added to that branch.
- Feature → develop merges are **autonomous** once verification + review pass. The `release/<version>` → `main` PR (a **release**) requires owner approval. Those are the only two human gates: feature design and release.

## Documents

| File | Owner | Purpose |
|---|---|---|
| `VISION.md` | The owner | What the product is and where it's going. Agents may propose edits when feature work surfaces new clarity, but show the owner the diff. |
| Work tracker | Agents | Queue of features/bugfixes. **Default: the git host's issues** (labels `ready-for-agent` → `in-progress` → `in-review` → closed; pick by `priority:` label then oldest-created; PR `Closes #N`). Fallback (no tracker): `BACKLOG.md` (Ready → In progress → Done / Icebox). Each item links its PRD. See the git-host adapter and `ORCHESTRATION.md`. |
| `docs/prds/NNNN-slug.md` | Agents | One PRD per feature, format in `.claude/autodev/PRD-FORMAT.md`. Includes the atomic task checklist and build log. |
| `CONTEXT.md` | Agents | Domain language and relationships. Update when terminology crystallises. |
| `docs/adr/NNNN-slug.md` | Agents | Decisions that are hard to reverse, surprising without context, and the result of a real trade-off. |

Keep these current as part of every loop — they are what lets the next session (or another person's agent) pick the project up cold.

## Stack discipline

- **The stack is whatever the project's stack preset says it is.** A new project scaffolds the preset's default shape; an existing project keeps its own stack. The methodology never assumes a particular language, framework, or cloud.
- **Existing projects keep their stack.** Respect a different architecture absolutely; suggest changes only if asked or if the request is literally impossible otherwise. Note mismatches with the preset's defaults in an ADR and move on — never migrate or restructure unprompted.
- **Preview/staging isolation is non-negotiable**, however the stack provides it: non-production deployments must use separate data stores and test-mode third-party credentials — never production data or live keys. New external bindings/resources are added to both environments in the same PR. The stack preset documents how this is achieved on that platform.
- Load any stack-specific skills the project has installed and bias to retrieving current docs over pre-trained knowledge for platform code.

## Production access & monitoring

- **Production is observability-only** (`config.prodTest`): the agent monitors prod via logs/errors/exceptions and **does not act as a user there** — no prod login. To exercise an app *as a user*, it uses **local** (the dev test account) and **staging** (the same dev login, enabled on staging only — see below). This avoids any test-account writes against real production data.
- **Staging dev login.** To let the loop log in to staging for the authenticated walkthrough, enable the project's dev test-account login on staging, **env-gated to `development OR staging`, never production** — one shared predicate at the auth gate, the seed endpoint, and the login form. The login path must bypass any CAPTCHA / OAuth / email-verification step (those gate other sign-in methods); the loop must never attempt to solve them. The dev fixture credentials are a public fixture, not a secret. If staging is network-gated, the loop passes the read-only access credential as headers to reach it.
- After a release: **close watch for ~1 hour** — poll observability for errors every few minutes and run one `/bug-hunt` pass — then **back off to a daily check** to save tokens. Critical findings become hotfixes immediately; everything else becomes a work-tracker item.
- Run `/refactor-pass` regularly — a conservative, behavior-preserving cleanup so codebase complexity never outgrows what an agent can navigate. In the autonomous loop the **release loop** owns it: a refactor pass runs **after each successful ship** (post-release cleanup is exactly what it's for), and as the release loop's idle action when there's nothing to ship or release. Rate-limited to ~once/day and only when there's new code since the last pass. Refactor lands on `develop` like any feature.

## Self-contained — bundled skills, not borrowed ones

Autodev is self-contained: every capability its commands rely on ships **inside the project**, installed under `.claude/skills/` at setup. The commands never depend on a skill that happens to be in a contributor's personal global collection, so the project behaves identically for anyone who clones it.

- **Grilling** is autodev's own bundled **`grill`** skill (`.claude/skills/grill/`), used by `/vision`. It interviews one question at a time, challenges against `VISION.md`/`CONTEXT.md`/ADRs, and updates docs inline.
- **Refactor target-finding** lives directly in `/refactor-pass` (it reads `CONTEXT.md`/ADRs and works a prioritised target list).
- **Production QA** lives directly in `/bug-hunt` (systematic journey-walking + edge probing).

The only genuinely external dependency is a **browser-automation tool** (used by both verification tiers): any tool that can drive a browser with custom request headers, persistent login, screenshots, and short recordings — named in `config.evidence.tool`. That's a *tool*, not a skill, and there's no single-vendor assumption.

Stack-specific skills installed **into the project** by a stack preset (e.g. a platform's official skills) are part of that preset and are shared with the repo — they're not borrowed from a contributor's personal collection either.
