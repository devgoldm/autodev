# Setup Reference

Universal details for the autodev-setup steps. Anything tied to a specific platform, git host, or automation runtime lives in the selected **preset** under [presets/](presets/) — this file stays neutral and points there. All shell examples are defaults — adapt to the project.

## Prerequisites

Check and report; offer to install what's missing rather than failing. These are the **universal** prerequisites; each selected preset lists its own on top.

- **A git-host CLI**, authenticated — needed for PRs and (if the host has them) issues + branch protection. GitHub's `gh` is the reference; see [presets/githost-github.md](presets/githost-github.md). If the project has no remote, offer to create one on the chosen host.
- **A test runner** for the stack (the stack preset names the default).
- **A browser-automation tool** on the host machine (e.g. Playwright, or any agent-driven browser) — used by the two testing tiers: the local build-step dogfood and the authenticated staging walkthrough. It needs to support setting request headers (to pass a staging network gate), persistent/seeded login, screenshots, and short screen recordings. Confirm it's installed and can launch a browser. Record it in `config.evidence.tool`.
- **A loop runtime** for the autonomous cascade — *your agent's own scheduled-automation feature* (Claude Code scheduled tasks/routines, Cursor automations, Codex scheduled runs, …; the worked reference is [presets/runtime-claude-code.md](presets/runtime-claude-code.md)) or plain cron/CI ([presets/runtime-cron-ci.md](presets/runtime-cron-ci.md)). Only needed if the owner wants the unattended loop; the interactive commands work without it.
- **No external helper skills.** Grilling, refactor target-finding, and production QA are all **bundled with autodev** — the `grill` skill (`templates/grill/`, installed to `.claude/skills/grill/`), the `thermo-nuclear-code-quality-review` skill (`templates/thermo-nuclear-code-quality-review/`, installed to `.claude/skills/thermo-nuclear-code-quality-review/` and invoked by `/refactor-pass`), and the `/refactor-pass` + `/bug-hunt` commands themselves. The project never relies on a skill from a contributor's personal global collection. (Do not install or substitute someone's own grilling/architecture/QA skills.)
- **Stack-specific prerequisites** — see the selected stack preset (its toolchain, CLIs, and credentials). For an existing project, detect what's already there instead of installing the preset's defaults.
- **Context budget** (if the harness supports it) — write (or merge into) the project's `.claude/settings.json` so sessions auto-compact at 20% of the context window:

```json
{
  "env": {
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "20"
  }
}
```

## New project scaffold

The concrete scaffold is **stack-specific** — follow the selected stack preset's § New project scaffold ([presets/stack-cloudflare.md](presets/stack-cloudflare.md) is the worked example; [presets/stack-generic.md](presets/stack-generic.md) is the fill-in template). Whatever the stack, the scaffold must end up with:

- A **test runner** wired and green on an example test.
- A **flags module** wrapping the chosen flag mechanism: a typed `isEnabled(env, "flag_name")`-style helper; evaluation failure or a missing flag = **off** (safe default). During setup, create one `example` flag and verify it evaluates end-to-end. (Skip only if `config.flags.mechanism: "none"` was deliberately chosen.)
- **Preview/staging isolation** from day one — a non-production environment with its **own** data stores and test-mode third-party credentials, never production data or live keys. How the stack achieves this is in its preset.
- If there's auth, a **test account** path: a seed script that creates/updates a user from `AUTODEV_TEST_EMAIL` / `AUTODEV_TEST_PASSWORD`, credentials stored in `.env` (gitignored). Record the env var names in `config.prodTest`.
- Strict type checking + a linter/formatter with sensible defaults, where the language supports them.

## Existing project onboarding

1. Explore the codebase thoroughly (use Explore subagents to keep tokens down).
2. Write `CONTEXT.md`: domain language, relationships, example dialogue, flagged ambiguities. (If a grilling skill with a context format is installed, follow its format.)
3. Seed `docs/adr/` with the 3–8 evidently load-bearing decisions already embodied in the code (stack choice, data model shape, integration patterns). Mark them as reverse-engineered.
4. Draft `VISION.md` from what the app observably does and walk the owner through it for correction — the owner owns this file.
5. Detect the existing stack, flag mechanism, test setup, and deploy pipeline; record them in `config.json` (write a short stack preset from `stack-generic.md` describing them) instead of imposing the defaults. Only scaffold what's genuinely missing (e.g. no flags mechanism at all).
6. **Never** restructure their architecture. Note mismatches with autodev defaults in an ADR and move on.

## Git & deploys

```bash
git init -b main                      # new projects only
git checkout -b develop main
git push -u origin main develop
```

- **Branch protection**: require PRs into `main` and `develop`; allow the agent to merge its own PRs once checks pass (no required human review on develop — the human gates are feature design and release). The exact commands are host-specific — see the git-host adapter ([presets/githost-github.md](presets/githost-github.md) § Branch protection). The intent is "PRs only, no force-push", not heavyweight review requirements.
- **Deploy pipeline**: connect whatever turns a merge into a deployment — `main` → production, `develop`/PRs → previews. This is **stack-specific** (the stack preset's § Deploy pipeline; e.g. the Cloudflare preset uses Workers Builds). If a hosted CI build can't be used, fall back to a documented deploy command run from `/release` and `/build-next` and record an ADR.
- **Preview isolation (required)**: non-production deployments must point at **separate** data stores and **test-mode** third-party credentials — never production data or live keys. When a feature later adds a binding/resource, add it to **both** environments in the same PR. Apply schema migrations to the non-production store first, then production at release. How the stack provides isolated environments is documented in its preset; existing projects with their own staging/preview convention keep it (record in `config.deploy`, and flag the one real gap worth fixing: previews pointing at production data).

## Notifications & approval

The autodev cascade ([templates/ORCHESTRATION.md](templates/ORCHESTRATION.md)) contacts the owner one-way and takes the **release approval as a lightweight signal** — no chat infrastructure. The exact mechanism is **runtime-specific**:

- **Claude Code runtime** (reference): notifications = built-in **Push**; approval = a **`release:approved` label** on the `release/*`→`main` PR (one tap on mobile). See [presets/runtime-claude-code.md](presets/runtime-claude-code.md).
- **Cron/CI runtime**: notifications = whatever channel the runtime can post to (a webhook, email, a CI annotation); approval = a label or a manual workflow-dispatch. See [presets/runtime-cron-ci.md](presets/runtime-cron-ci.md).

Whatever the runtime: the loops are **silent on idle** and signal only when a release PR is ready to approve or a build escalates; bug-hunt notifies once per run. The approval signal's **trust boundary is repo write access** — anyone who can produce it can ship, so keep collaborators limited to people trusted to release. Create the approval label + the lifecycle labels (`in-progress`, `in-review`, `parked`, `priority:high|med|low`) once during setup via the git-host adapter. Record the model in `config.loop`.

## The autonomous loop

The cascade runs as **three scheduled tasks** registered with the chosen **loop runtime** (see the runtime preset for where they live and how to schedule them). It is **not** required for the interactive commands; it is what makes the project run unattended.

Create all three from `templates/scheduled-tasks/`, filling every `<PLACEHOLDER>` with this project's values:

- **`autodev-build`** — hourly, self-notifies on escalation. Build-only: **atomically claim** one `ready-for-agent` work item (see the git-host adapter's *Claiming work* section), build it in a worktree, verify locally, PR into the integration branch (auto-merge), label `in-review`. **Scale freely:** run as many build loops as you want — on extra machines and/or several on one machine (default `config.loop.build.instancesPerMachine: 1`, which keeps token spend predictable). Each loop is a separate `autodev-build` task with a distinct **worker suffix** + **dev port**, on a **staggered** schedule so claims rarely contend. There is **no global build lock** — the per-loop claim (keyed on worker id, not machine) is the only concurrency control.
- **`autodev-release`** — daily, **primary machine/host only** (it owns prod monitoring + the approval gate). Per run, precedence **Ship → Release-ask → Refactor(idle)**: Ship merges the frozen `release/*`→`main` if approved, then flips prod flags + close-watches + back-merges `main`→`develop` + runs a refactor pass; else Release-ask cuts+freezes a `release/<version>` branch, opens the PR, runs the authoritative review + staging evidence, and notifies; else a rate-limited idle `/refactor-pass`.
- **`autodev-bug-hunt`** — daily, **primary**. Logs-only prod triage → opens `ready-for-agent` bug items (feeds the build loop).

Things the loops depend on, set up here (runtime-agnostic):

- **Worktree isolation (hard invariant).** The host's live checkout is a **read-only base** — a loop never switches its branch or edits its tree. The build/refactor steps work in an ephemeral worktree under `.claude/worktrees/`, reusing installed dependencies, and remove the worktree + local branch as soon as the PR is pushed. This is what lets the loops run on a machine someone also works on.
- **Two-tier testing.** *Local* (every build, before the integration merge): start the stack, seed, dogfood the feature with the browser-automation tool + tests/typecheck/lint; green-or-park; no artifacts; auto-merge on green CI. *Staging* (every release): authenticated walkthrough (screenshots + a short recording per shipped item) **shown in the release run's transcript** — never committed or hosted. Flags ship **on in staging / off in production**; the release flips production flags on.
- **Staging auth (for the authenticated walkthrough).** Enable the project's dev test-account login on staging, env-gated to `development OR staging`, **never production** (one shared predicate at the auth gate + the seed endpoint + the login form). The login must skip any CAPTCHA / OAuth / email step — those gate other sign-in methods, not the dev password path; the loop must never attempt to solve them. If staging is behind a network gate, put the read-only access token in the host's gitignored env so the browser tool can pass it as headers. The dev fixture credentials are not secrets. Skip this only if you accept public-surface-only staging evidence.

The owner's **front door (grilling) is NOT a scheduled task** — it is a live, owner-initiated interactive session so `AskUserQuestion` works. The loop handles everything after the push; only the release ask and escalations reach the owner.

## config.json reference

`.claude/autodev/config.json` — filled at setup, read by every command. The `*Preset` fields point at the selected preset files under `.claude/autodev/presets/`.

```json
{
  "owner": { "technicalLevel": "technical | semi-technical | non-technical" },
  "stack": { "preset": ".claude/autodev/presets/stack-cloudflare.md", "describe": "worker+hono+vite-react-spa", "respectExisting": true },
  "gitHost": { "preset": ".claude/autodev/presets/githost-github.md", "cli": "gh", "repoSlug": "owner/repo" },
  "flags": { "mechanism": "flagship | kv | launchdarkly | env | none | existing:<describe>", "stagingOnProdOff": true },
  "deploy": { "mechanism": "ci-on-merge | manual | existing:<describe>", "branchingModel": "gitflow", "productionBranch": "main", "integrationBranch": "develop", "releaseBranchPrefix": "release/", "releaseVersionScheme": "date (release/YYYY-MM-DD, suffix -2/-3) while unversioned", "backMerge": "after every ship, back-merge main->develop via a merge-commit PR", "productionUrl": "", "previewEnv": "preview", "previewUrl": "" },
  "prodTest": { "method": "observability-only", "observability": "<how prod logs/errors are read>", "loginUrl": "", "envVars": [] },
  "staging": { "url": "", "healthCheck": "<how to confirm the integration deploy is green>", "access": "none | network-gate-token", "accessEnvVars": [], "devLogin": { "email": "", "password": "" } },
  "evidence": { "host": "local-only-shown-in-transcript", "tool": "<browser-automation tool>", "captureDir": "mktemp -d" },
  "loop": {
    "runtime": ".claude/autodev/presets/runtime-claude-code.md",
    "doc": ".claude/autodev/ORCHESTRATION.md",
    "tasks": ["autodev-build", "autodev-release", "autodev-bug-hunt"],
    "host": "<primary machine>; build loop may also run on extra machines",
    "frontDoor": "named-session | local-session",
    "notify": "<runtime notification channel, e.g. push>",
    "approval": "<runtime approval signal, e.g. label:release:approved>",
    "build": { "instancesPerMachine": 1, "everyMinutes": 60 },
    "claim": { "mechanism": "claim-comment", "marker": "🤖 autodev-claim", "workerFrom": "$(scutil --get LocalHostName || hostname -s)<WORKER_SUFFIX>", "claimTtlHours": 3, "tiebreak": "createdAt-then-worker" },
    "release": { "host": "primary-only" }
  },
  "workTracker": { "type": "host-issues | backlog-file", "readyLabel": "ready-for-agent", "priority": "label-then-oldest", "prdsAsFiles": true },
  "models": { "subagents": "<most capable available model>" },
  "escalation": { "maxFixCycles": 3 },
  "monitoring": { "closeWatchMinutes": 60, "pollSeconds": 270, "steadyState": "daily" }
}
```

Preset files referenced here are also documented for the project under `.claude/autodev/presets/`. Any preset-specific extras (e.g. a Cloudflare account ID + token recipe) live in that preset's own block and/or a copied artifact such as `.claude/autodev/cloudflare-token.md`.
