---
name: autopilot-tickets
description: Continuously drain ready-for-agent implementation tickets or issues by repeatedly dispatching safe batches to subagents, waiting for draft PRs or MRs, verifying merge gates, merging only safe agent-created PRs or MRs, refreshing ticket state, and continuing until no ready-for-agent tickets remain. Use when the user explicitly asks Codex or Claude Code to run an automatic ticket-processing loop, auto-merge completed agent PRs/MRs, or keep dispatching ready ticket work until the queue is empty.
---

# Autopilot Tickets

## Ownership

- `dispatch-tickets` owns per-ticket readiness, branch/worktree setup, `implement` subagent prompting, and subagent lifecycle.
- `autopilot-tickets` owns fresh-state looping, blocker detection, draft promotion, merge gates, and stopping.

Use configured tracker/forge tools. Delegate to the `dispatch-tickets` skill; if unavailable, manually apply its readiness/lifecycle rules and report the fallback.
Only accept or create ticket worktrees that match the ledger path `<project-root>/.worktrees/<branch-name>`.

## Loop

Repeat from fresh state:

Explicit invocation authorizes safe in-scope recovery and merge without repeated confirmation.
Replace unavailable workers, send targeted follow-ups, and repair loop-created changes within the same ticket scope autonomously.

1. Run `git status --short --branch`; fetch, identify the base branch, and resolve the repo root.
2. Query open `ready-for-agent` tickets, tracked PRs/MRs, labels, links, project fields, milestones, comments, dependencies, and blockers.
3. Rebuild global dependencies/blockers; leave per-ticket dispatchability to `dispatch-tickets`.
4. Process tracked PRs/MRs first and merge only after every gate passes. If none is mergeable, call the `dispatch-tickets` skill for all safe tickets without a fixed skill-level concurrency limit.
5. Wait for all dispatch results. Only the parent may promote a verified draft; refresh after promotion, then merge only if every gate still passes.
6. After each merge, confirm the linked ticket closed or updated and refresh tracker, dependencies, and blockers before evaluating another PR/MR or dispatching more work. If nothing merges, refresh before another round.

Foundational/shared-contract work must merge and appear in the refreshed state before dispatching dependents.

## Merge Gates

Merge only PRs/MRs that pass every gate:

- Created by this loop or a tracked prior round; linked to its assigned ticket.
- Source branch and exact worktree match the dispatch ledger.
- Parent verified `DONE`, branch, commit, changed-file scope, PR/MR description, and tests.
- Parent promoted the verified draft to ready-for-review; current diff remains within acceptance criteria.
- Required checks pass; no checks are failed.
- No unresolved review comments, merge conflicts, requested human decisions, new blocker labels/comments, or dependency changes since dispatch.
- No repo rule, branch protection, missing approval, or policy blocks merge.

Use the repo's normal merge method; do not invent squash/rebase/merge policy.
Do not auto-merge broadened scope, human-authored changes, production secrets, deployment controls, destructive migrations/data changes, payments, auth/access policy, or legal/compliance text without explicit authorization and all required reviews.

## Stop And Report

Stop under any condition below; record the blocker rather than dispatching dependent or adjacent work:

- No open `ready-for-agent` implementation ticket remains, or none can be safely dispatched.
- Foundational/shared-contract work is blocked/awaiting review, or a failed merge gate blocks dependents.
- Forge, tracker, git, or subagent tools are unavailable, or failed tests/checks resist safe targeted follow-up.
- The repo has dirty/conflicting state the loop did not create.
- User input, repo policy, branch protection, or review requirements need human decision.

Carry forward the dispatch ledger and add:

- Round number and draft/ready state.
- Current verification commands/results, gate result, merge SHA or reason not merged, and linked-ticket closure.
- Undispatched ready tickets grouped by reason, exact stop condition, and next-loop conditions.
