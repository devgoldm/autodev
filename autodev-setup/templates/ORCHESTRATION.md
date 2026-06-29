# Autodev orchestration

> Status: fill in during setup. Project-specific layer over [METHODOLOGY.md](METHODOLOGY.md):
> *how an idea becomes production* via an autonomous scheduled-task loop with a conversational front door.
>
> Setup note (delete after filling in): set the repo path, integration/production branch names, the
> release-branch prefix, the staging URL, and the front-door session name below; name the selected
> **presets** (stack / git-host / loop runtime); and confirm the three scheduled tasks under "Still to
> set up" were actually created (build on each machine, release + bug-hunt on the primary).
>
> Presets for this project (under `.claude/autodev/presets/`):
> - Stack: `<stack preset>` · Git host: `<git-host adapter>` · Loop runtime: `<loop runtime>`
> Read those for the concrete commands wherever this doc refers to a role (git host, deploy pipeline,
> flag mechanism, loop runtime).

## Principle

The owner's front door is a **conversation, in the moment**, not a file. They say what they want
("I want feature X", "bug Y appeared"), get **grilled there and then**, and that live session turns
it into a queued, pushed unit of work. An autonomous scheduled-task loop carries it the rest of the
way. The owner is contacted again **only when it is ready to ship**, and ships with one approval.

Two touchpoints per item:

1. **Bring an idea + get grilled** — owner-initiated, live, immediate. No notification (the owner
   is already there), and `AskUserQuestion` works because the session is interactive.
2. **Release approval** — the runtime's approval signal on the `release/<version>`→production PR
   (the reference runtime: add the `release:approved` label — one tap on mobile).

`VISION.md` still exists as the living north-star, but it is **maintained by the grill as a
byproduct**, not the thing the owner edits to start work.

## Loop runtime

The cascade runs as **three scheduled tasks** registered with the project's **loop runtime** (see the
runtime preset). The reference runtime is Claude Code scheduled tasks under `~/.claude/scheduled-tasks/`,
ticking while the app is open; the cron/CI runtime runs the same tasks unattended. The runtime governs
only the scheduler, the notification channel, and the approval signal — everything below is the same
regardless.

## Work tracking — the work tracker (default: host issues)

**Default: the work queue is the git host's issues** (see the git-host adapter), not a file. Prefer
this whenever the repo already has an issue tracker — commentable from mobile, links cleanly to PRs.
**Retire `BACKLOG.md`** so there is a single source of truth.

- **One issue = one unit of work** (feature or bugfix). The grill opens it.
- **The PRD stays a file** (`docs/prds/NNNN-slug.md`), versioned and reviewed in the PR; the issue **links** it.
- **Status = labels**: `ready-for-agent` → `in-progress` → `in-review` → **closed** (shipped);
  `parked` / `wontfix` for parked/dropped.
- **Pick order**: by **`priority:` label** first, then **oldest-created** within the same priority.
- **PR closes the issue** (`Closes #N`).

**File fallback:** if the repo has *no* issue tracker, keep `BACKLOG.md` as the queue instead
(`Ready` → `In progress` → `Done` / `Icebox`, each `Ready` item linking its PRD, top item first).
Record which model the project uses in `config.workTracker`.

## The loops

| Loop | Task | Schedule | Machines/hosts | What it does |
|---|---|---|---|---|
| **Build** | `autodev-build` | hourly | **any/many** (staggered) | Atomically **claim** one top `ready-for-agent` item → build in a worktree → verify locally → PR into the integration branch with `Closes #N` (auto-merge `--squash` on green CI) → `in-review`. Also: stuck-CI → `parked`; stale-claim reaper. |
| **Release** | `autodev-release` | daily (a fixed time) | **primary only** | `Ship` (merge the frozen `release/*`→production if approved) → on success flag-flip + prod close-watch + back-merge to integration **then a refactor pass**; else `Release-ask` (cut+freeze `release/<version>`, authoritative review, staging evidence); else idle-`Refactor` (rate-limited). |
| **Bug-hunt** | `autodev-bug-hunt` | daily | primary only | Logs-only production triage via the observability source; opens `ready-for-agent` bug items (which feed the build loop). |

The build loop **scales horizontally** — add a machine by creating `autodev-build` there on a
different schedule; the atomic claim (below) keeps two machines from ever building the same issue.
The release + bug-hunt loops run on the **primary** host only.

### Gitflow & the frozen release branch

```
feature/<slug>  →  develop  →  release/<version>  →  main (production)
                                      └──────── back-merge ───────→ develop
hotfix/<slug>   →  main  (+ back-merge → develop)
```

- **Build** merges `feature/*` → the integration branch (squash) continuously — this is **never frozen**.
- **Release-ask** cuts `release/<version>` off the current integration tip (the freeze point) and
  opens the PR **`release/<version>` → production**. Build's later merges to the integration branch
  do **not** touch this branch, so the open release PR's diff is **frozen** — the review is
  **authoritative**, and the flag plan is derived once from the frozen branch.
- Stabilization fixes found in review/staging are committed **on the release branch** (so reviewed
  set === shipped set).
- **Ship** merges `release/<version>` → production (**merge commit**, never squash — squashing makes
  production stop being an ancestor of the integration branch), then **back-merges production → the
  integration branch** (a `chore/sync-...` PR, merge commit). **Hotfixes** branch off production, PR
  back, and back-merge the same way.
- **Version scheme:** default date-based `release/YYYY-MM-DD` (suffix `-2`/`-3` for multiple per day)
  while a product is pre-launch/unversioned; swap for semver later.

### Claiming work (atomic)

The build loop has **no global build lock** — each build loop claims a single issue atomically, so N
loops each hold a different one. Coordination is by **worker id** (the machine's hostname plus an
optional suffix), not by machine, so you can run several build loops on **one** machine as well as
across many. The full protocol (stale-claim reaper, self-recovery guard, claim, server-timestamp
tiebreak, loser-deletes-claim) lives in the **git-host adapter's *Claiming work* section** because it
depends on the host's commands. Summary: relabel `ready→in-progress`, post a
`🤖 autodev-claim worker=<worker-id>` comment, wait ~12s, and the winner is the **earliest
server-side `createdAt`** (immune to local clock skew), ties broken by lexical worker id; losers
delete their own claim comment and try the next issue. The self-recovery guard keys on the worker id,
so loops on the same machine don't block each other. Staggering schedules makes contention near-zero;
the protocol is the correctness backstop.

**Isolation & cleanup (the hard invariant):** the contributor's live checkout is a **read-only
base** — a loop never switches its branch or edits its working tree. The build/refactor steps do all
code work in an **ephemeral git worktree** under `.claude/worktrees/` (branch off the integration
branch), reusing the main repo's installed dependencies (only a real install if the lockfile
differs). As soon as the PR is pushed it removes the worktree **and** deletes the local branch ref,
so disk and the local branch list stay clean.

**No deploys from the machine.** The local monitoring credential is **read-only**; deploys happen
only through the deploy pipeline on merge to production. `/release` only merges the PR and watches CI.

## Testing & evidence

Nothing reaches production un-exercised. Two tiers, deliberately different:

- **Local — every build, before it merges to the integration branch.** Inside the build worktree
  the **build loop** starts the full stack, seeds test data, and exercises the new feature with the
  browser-automation tool alongside the test suite + typecheck + lint. It only opens the PR if all of
  that is green; otherwise the issue is `parked`. **No artifacts kept.** The build PR then auto-merges
  into the integration branch once CI passes.
- **Staging — every release, before the owner is asked to approve.** Because flags ship **on in
  staging / off in production**, the staging deploy actually runs the feature. The **release loop**
  (Release-ask) drives the staging URL with the browser-automation tool (logged in as the dev test
  account — see [Staging auth](#staging-auth)), and per shipped `Closes #N` records a short
  walkthrough + a few screenshots. **Evidence is never committed and never externally hosted.** The
  release run **displays the key screenshots inline at the end of its run** so they render in the run
  transcript; the release PR body just **notes** that evidence is in that run.

(If the project has no flag mechanism or no staging environment, this collapses to local verification
+ post-release monitoring — see METHODOLOGY § Two-tier verification.)

## Staging auth

For the staging walkthrough the loop needs a logged-in session, without solving a CAPTCHA / OAuth /
email step (it must never attempt those). The clean pattern: **enable the same dev test-account login
on staging that you use locally**, env-gated to non-production only — a single predicate
`development OR staging`, **never production**. If staging sits behind a network gate, the loop passes
the access-token headers to reach it, then logs in with the dev fixture. The test-account credentials
are a well-known fixture, not a secret. If no such login exists, staging evidence falls back to
public-surface + health, and the authenticated proof comes from the local build-step run.

## Notifications & approval

Runtime-specific — see the loop runtime preset.

- **Notifications**: the loops are **silent on idle**; a notification is sent only when a release PR
  is ready to approve or a build escalates; bug-hunt notifies once per run. Most channels are
  text-only — they point at the PR / run, not carry images.
- **Release approval**: the runtime's approval signal on the `release/<version>`→production PR (the
  reference runtime: the `release:approved` label). The next release run sees it and ships. The trust
  boundary is **repo write access** — anyone who can produce the signal can ship, so keep
  collaborators limited to people trusted to release.

## Lifecycle

| # | Stage | Who | Trigger | Produces | Owner contact |
|---|---|---|---|---|---|
| 1 | **Grill (front door)** | Owner, live | Owner says "I want X" / "bug Y" | Grills (`/vision`); updates `VISION.md` if warranted; writes the PRD; opens a **`ready-for-agent` item** linking the PRD (with a `priority:` label); pushes | None (owner is there) |
| 2 | **Build** | `autodev-build` (any machine) | A `ready-for-agent` item exists | **Atomically claims** the top item; builds in an ephemeral worktree (flag on-in-staging/off-in-prod); **verifies locally**; PR `Closes #N` into the integration branch, auto-merged on green CI → staging deploy | Notify only on escalation |
| 3 | **Release-ask** | `autodev-release` (primary) | Integration branch ahead of production, no open release PR | Staging health; **cuts + freezes `release/<version>`**; opens the `release/*`→production PR with the flag plan; **authoritative review = security review + a general engineering review** over the frozen diff; **captures authenticated staging evidence** (shown in the run) | **Notify: "RELEASE PR #N READY — approve to ship"** |
| 4 | **Release-ship** | `autodev-release` (primary) | Approval signal on the `release/*`→production PR | Light delta re-check; `/release`: merge to production (merge commit, flags flipped on), close-watch; **back-merge production→integration**; refactor pass; items close on merge | **Notify: "SHIPPED — prod healthy"** |
| 5 | **Refactor** | `autodev-release` (primary) | After each ship, or release loop idle; rate-limited | Conservative `/refactor-pass` in a worktree → PR into the integration branch (auto-merged on green CI) | Silent (notify only if it can't stay test-green) |
| 6 | **Bug-hunt** | `autodev-bug-hunt` (primary) | Daily | Logs-only triage → opens `ready-for-agent` bug items (→ feeds Build) | Notify once per run |
| — | **Escalation** | `autodev-build` | A build fails its cycle budget | Item labelled `parked` with a note; worktree cleaned up; no silent retries | **Notify: "🚨 STUCK …"** |

## Release approval in detail (Gate 2)

1. A release run does **Release-ask**: cuts the frozen `release/<version>` branch, opens the
   `release/*`→production PR, and notifies "RELEASE PR #N READY".
2. The owner produces the approval signal on that PR (the reference runtime: adds `release:approved`).
3. The next release run does **Ship**: it guards (production base + approval present), does a light
   delta re-check on any stabilization commits, then runs `/release` — merge, deploy, close-watch,
   back-merge to the integration branch, refactor.

## Still to set up / confirm

- The three scheduled tasks created via the loop runtime from the templates (filled with this repo's
  path + branch/URL specifics): `autodev-build` on **each** build machine at a **staggered** schedule;
  `autodev-release` + `autodev-bug-hunt` on the **primary** host only.
- The lifecycle labels (`in-progress`, `in-review`, `parked`, `priority:high|med|low`) and the
  release approval signal created once (see the git-host adapter + loop runtime preset).
- A **named front-door session** on the host machine for the live grill, if wanted.
- The read-only monitoring credential in the host's gitignored env; and, if staging is gated, the
  staging access credential there too.
- The staging dev-login enablement (see [Staging auth](#staging-auth)) if authenticated staging
  evidence is wanted.
