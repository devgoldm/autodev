# PRD Format

PRDs live in `docs/prds/` as `NNNN-slug.md` (sequential — scan for the highest number and increment). One PRD per feature, written by `/vision`, executed and updated by `/build-next`.

## Template

```md
---
status: ready            # ready | building | built | shipped | parked
flag: feature_slug       # the feature flag gating this work
---

# {Feature name}

## Problem Statement

The problem the user is facing, from the user's perspective.

## Solution

The solution, from the user's perspective.

## User Stories

A numbered list covering all aspects of the feature:

1. As an <actor>, I want <feature>, so that <benefit>

## Implementation Decisions

Decisions from the grilling session: modules built/modified and their interfaces, architectural decisions, schema changes, API contracts, specific interactions. Do NOT include file paths or code snippets — they go stale. (Exception: a snippet that encodes a decision more precisely than prose — a schema, state machine, type shape.)

## Testing Decisions

What will be tested and how: external behavior only, not implementation details. Name the modules under test and prior art for similar tests in the codebase.

## Out of Scope

What this PRD deliberately does not cover.

## Tasks

Atomic, independently verifiable tasks, each completable and verifiable by one subagent in one sitting. Ordered as tracer-bullet vertical slices: the thinnest end-to-end path first, then flesh out.

- [ ] 1. {task — concrete enough that a cold agent can do it from this line + the PRD}
- [ ] 2. ...

## Build log

Appended by /build-next as work proceeds: per-task verification notes, surprises, failed approaches, and — if parked — exactly what was tried and why it failed.
```

## Rules

- Status moves `ready → building → built → shipped`; `parked` from anywhere, with a Build log entry explaining why.
- Tasks are the contract with build subagents: if a task can't be verified on its own, split or reorder it.
- Update the PRD as decisions change during the build — it should read true after the fact, not just before.
