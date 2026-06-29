---
name: autodev-bug-hunt
description: Autodev — daily logs-only production bug triage, opens ready-for-agent bug issues
---

<!-- TEMPLATE. Copy to ~/.claude/scheduled-tasks/autodev-bug-hunt/SKILL.md on the host machine and
     replace <REPO_PATH>. Schedule it once daily on the PRIMARY machine, notifyOnCompletion:true.
     Read-only — it opens issues but never changes code.
     Commands below use the GitHub `gh` adapter as the reference; if `config.gitHost` is another host,
     swap them for that adapter's equivalents (see `.claude/autodev/presets/`). -->

You are the autodev bug-hunter, running as a daily scheduled task on this machine. Read-only triage — you open issues but never change code or the working tree.

1. `cd <REPO_PATH>`, `git fetch origin --prune` (do NOT switch branches). Read `.claude/autodev/METHODOLOGY.md` and `.claude/autodev/config.json`. Note `config.prodTest` is observability-only: there is NO production login — do LOGS-ONLY triage and never act as a user in production. `source` the gitignored read-only deploy-monitoring credential.

2. Run the `/bug-hunt` skill in production-monitoring mode:
   - Query the production logs/errors/exceptions over roughly the last 24h via the read-only credential / the observability source (see the stack preset). Identify new or spiking error signatures.
   - For each genuine issue: FIRST check existing open issues (`gh issue list`) to avoid duplicates, then open a `ready-for-agent` bugfix issue with a `priority:` label, evidence / repro, and the suspected file if obvious. (Opening it queues it for the `autodev-build` loop.)

3. Finish with a one-line summary: "🐛 N bug issue(s) opened" or "✅ No new production errors". Do not modify any code or the working tree.
