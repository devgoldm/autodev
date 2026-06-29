---
name: bug-hunt
description: Hunt for bugs in production — check the observability source for errors, log in with the safe test account, exercise the app non-destructively, and triage findings into hotfixes or backlog bugfixes. Use when the user runs /bug-hunt, asks to check on production, QA the live app, or investigate errors.
---

# Bug Hunt

Read `.claude/autodev/METHODOLOGY.md` and `.claude/autodev/config.json` first (and the selected presets under `.claude/autodev/presets/`).

## Hard safety rules

- Authenticate **only** via `config.prodTest` (test account credentials from the env vars listed there, or the Access service token). Never another account.
- **Non-destructive only**: no deletes, no irreversible mutations, no admin actions, never read or modify real-user data. Creating clearly-test-labeled data under the test account is fine; clean it up afterwards if the app allows.
- If real payments are reachable, use test-mode third-party credentials (e.g. Stripe test mode if the stack preset uses Stripe) only — never trigger a live charge.

## Steps

1. **Logs first**: query the observability source (see the stack preset / `config.prodTest.observability`) for the production service — errors, exceptions, status-5xx, unusual patterns since the last release (or last hunt). Each distinct error signature is a finding.

2. **Exercise the app**: log in to `config.prodTest.loginUrl` as the test account (browser tools if available, otherwise API calls). Walk the main user journeys systematically, prioritising recently released features (check recently closed work items / recent PRDs): for each journey, drive the happy path then probe the edges — empty states, invalid input, permissions, back/refresh/double-submit, and anything the PRD's acceptance criteria call out. Capture reproduction steps and evidence (screenshots, the failing request/response) for everything broken, wrong, or confusing.

3. **Triage each finding**:
   - **Critical** (data loss, auth broken, core journey dead, error spike): contain first — if a feature flag covers it, flip it OFF yourself via the flag API (owner standing order to manage flags via the API; if `config.flags.mechanism` is `none`, go straight to the hotfix). Then hotfix per METHODOLOGY: branch off `main`, fix via the build/verify subagent loop (`config.models.subagents`) with a regression test, PR to main, `/security-review`, merge, verify fixed in production, back-merge to develop. Notify the owner if user impact is meaningful.
   - **Non-critical bug**: open a `ready-for-agent` bug work item (git-host adapter; repro steps inline; no PRD needed for small fixes). Where `BACKLOG.md` is the tracker, add it under Ready.
   - **UX/idea**: open a work item labelled `icebox` (git-host adapter) (or add to `BACKLOG.md` Icebox) for the next `/vision` session to consider.

4. **Report**: production health verdict, findings with severity and what was done about each, log evidence. If the hunt was clean, say so plainly.
