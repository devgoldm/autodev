# Git-host adapter — GitHub

The reference **git-host adapter**: GitHub via the `gh` CLI, with GitHub Issues as the work tracker. The methodology refers to these by role (the *git host*, the *work tracker*, *issues*, *labels*, *PRs*); this file supplies the concrete commands. Another host (GitLab, Gitea, Forgejo…) needs an equivalent adapter — keep the **concepts** and swap the commands.

Record `config.gitHost.preset` = this file, `config.gitHost.cli` = `gh`, `config.gitHost.repoSlug` = `owner/repo`.

## Prerequisites

- **`gh`** authenticated (`gh auth status`). Needed for PRs, issues, labels, and branch protection.
- If the project has no GitHub remote, offer to create one: `gh repo create`.

## Work tracker — Issues

`config.workTracker.type` = `"host-issues"`. One issue = one unit of work (feature or bugfix).

- **Status = labels**: `ready-for-agent` → `in-progress` → `in-review` → **closed** (shipped); `parked` / `wontfix` for parked/dropped.
- **Priority labels**: `priority:high` → `priority:med` → `priority:low`.
- **Pick order**: by `priority:` label first, then **oldest-created** within the same priority.
- **PR closes the issue**: `Closes #N` in the PR body. (Note: `Closes #N` fires when the PR's *base* branch receives the commit — so a PR into `develop` closes the issue at merge, but if your default branch is production the close happens at the release merge. The build loop keeps the issue `in-review` until then.)
- The **PRD stays a file** (`docs/prds/NNNN-slug.md`), versioned and reviewed in the PR; the issue **links** it.

Create the labels once during setup:

```bash
gh label create ready-for-agent --color 0e8a16
gh label create in-progress     --color fbca04
gh label create in-review       --color 1d76db
gh label create parked          --color b60205
for p in high med low; do gh label create "priority:$p"; done
# release approval label (see the loop runtime preset):
gh label create release:approved --color 5319e7
```

**File fallback:** if the repo has *no* issue tracker, set `config.workTracker.type` = `"backlog-file"` and use `BACKLOG.md` (`Ready` → `In progress` → `Done` / `Icebox`, each `Ready` item linking its PRD, top item first).

## Common commands

| Action | Command |
|---|---|
| List ready work | `gh issue list --label ready-for-agent --json number,title,labels,createdAt` |
| Relabel | `gh issue edit <N> --add-label in-progress --remove-label ready-for-agent` |
| Comment | `gh issue comment <N> --body "…"` |
| Open PR | `gh pr create --base <branch> --head <branch> --title "…" --body "…Closes #N"` |
| Auto-merge on green | `gh pr merge <pr> --squash --auto` (feature→integration) / `gh pr merge <pr> --merge` (release→production) |
| Read PR checks | `gh pr checks <pr>` / `gh pr view <pr> --json statusCheckRollup` |
| Open release PRs | `gh pr list --base <production> --state open --json number,title,labels,headRefName` |

## Branch protection

Require PRs into `main` and `develop`; let the agent merge its own PRs once checks pass (no required human review on `develop`). The intent is "PRs only, no force-push", not heavyweight review.

```bash
gh api -X PUT "repos/{owner}/{repo}/branches/main/protection" --input - <<'JSON'
{ "required_pull_request_reviews": null, "required_status_checks": null,
  "enforce_admins": false, "restrictions": null,
  "allow_force_pushes": false, "allow_deletions": false }
JSON
```

(Adjust per current GitHub API; same for `develop`.)

## Claiming work (atomic, multi-loop)

The build loop has **no global build lock** — each build loop claims a single issue atomically, so N loops each hold a different one. The claim identity is a **worker id** — the machine's hostname plus an optional suffix — so you can run several build loops on **one** machine (each with a distinct suffix), not just one per machine. A **claim comment** is a comment whose body is exactly `🤖 autodev-claim worker=<WORKER>` (`config.loop.claim`; `claimTtlHours` default 3).

1. **Stale-claim reaper (first):** for each open `in-progress` issue, read its newest `🤖 autodev-claim` comment. If it is older than `claimTtlHours` **and** the issue has no open PR (`Closes #N`) **and** it isn't `parked`/`in-review`, relabel it back to `ready-for-agent` (comment "♻️ reclaiming stale build").
2. **Self-recovery:** if **this worker id** already owns an unfinished claim (its claim comment, no PR yet), skip claiming a new one this run. (Keyed on the worker id, so other loops — including others on the same machine — are unaffected.)
3. **Claim:** pick the top `ready-for-agent` (priority label, then oldest-created). `gh issue edit N --add-label in-progress --remove-label ready-for-agent`, then post `🤖 autodev-claim worker=<WORKER>`.
4. **Resolve:** sleep ~12s, fetch **all** `autodev-claim` comments on the issue (`gh issue view <N> --json comments`). **Winner = earliest GitHub `createdAt`** (authoritative server time, immune to local clock skew), ties broken by lexical worker id. If this worker is **not** the winner: delete its own claim comment (`gh api -X DELETE /repos/<slug>/issues/comments/<id>` — id is the digits after `#issuecomment-` in the comment `url`), leave labels alone (the winner keeps `in-progress`), and try the **next** `ready-for-agent` issue (bounded to a few retries, then idle).
5. **Build** the won issue. One build per loop per run.

Staggering loops onto different schedules makes real contention near-zero; the protocol is the correctness backstop. The **worker id** = `<hostname><suffix>`, where hostname is `scutil --get LocalHostName 2>/dev/null || hostname -s` (works on macOS and Linux) and the suffix is empty for a machine's default loop or `-2`/`-3`/… for additional loops on that same machine.

## Adapting to another host

A different host needs the same five capabilities: **list/label/comment on work items**, **open + auto-merge PRs**, **read PR check status**, **server-authoritative timestamps on claim comments** (for the tiebreak), and **branch protection**. GitLab maps issues↔issues, labels↔labels, PRs↔merge-requests, `gh`↔`glab`; the claim tiebreak needs a server-side `created_at` on notes. Write the equivalent commands into a `githost-<name>.md` and point `config.gitHost.preset` at it.
