---
name: autopilot-tickets
description: Drain ready-for-agent implementation tickets by dispatching safe batches to subagents, waiting for draft PRs/MRs, verifying merge gates, merging only safe agent-created PRs/MRs, and looping until the queue is empty or blocked. Use when the user explicitly asks for an automatic ticket-processing loop, auto-merge of completed agent PRs/MRs, or continuous ready-ticket dispatch.
---

# Autopilot Tickets

## Ownership

- `dispatch-tickets` owns per-ticket readiness, branch/worktree setup, worker prompting, and subagent lifecycle.
- `autopilot-tickets` owns fresh-state looping, blocker detection, draft promotion, merge gates, and stopping.

Use configured tracker/forge tools. Delegate to the `dispatch-tickets` skill; if unavailable, manually apply its readiness/lifecycle rules and report the fallback.
Subagent dispatch is delegated to `dispatch-tickets`. If the harness cannot dispatch subagents, stop the loop after reporting the missing capability — do not simulate workers in the parent session unless the user explicitly requests fallback inline work.

## Loop

Repeat from fresh state:

Invoking this skill grants standing authorization for the full run to promote and immediately merge every in-scope PR/MR once all Merge Gates pass. Execute the merge without requesting per-PR confirmation. This authorization does not waive Merge Gates or authorize out-of-scope changes.
Replace unavailable workers, send targeted follow-ups, and repair loop-created changes within the same ticket scope autonomously.

1. Run `git status --short --branch`; fetch, identify the base branch, and resolve the repo root.
2. Query open `ready-for-agent` tickets within the requested scope and PRs/MRs tracked by this run. Read their acceptance criteria, labels, recent comments, and linked dependencies/blockers; consult other tracker fields only when they affect readiness or merge gates.
3. Refresh dependencies/blockers for those candidates and tracked PRs/MRs, following linked prerequisites as needed. Treat a parent Spec/PRD/map link as scope, not a blocker: its open state never gates a child implementation ticket. Only an unfinished implementation-level dependency, an explicit human decision, or a real merge/review/policy gate blocks dispatch; leave per-ticket dispatchability to `dispatch-tickets`.
4. Process tracked PRs/MRs first and merge only after every gate passes. If none is mergeable, call the `dispatch-tickets` skill for all safe tickets without a fixed skill-level concurrency limit.
5. Collect all dispatch results using the available wait/status tools. While workers are running, use bounded waits instead of repeatedly re-querying unchanged tracker state. Only the parent may promote a verified draft; refresh after promotion, then merge only if every gate still passes.
6. After each merge, confirm the linked ticket closed or updated, then remove its exact clean ledger worktree with `git worktree remove` after worker activity ends; record failure instead of forcing removal. Refresh tracker, dependencies, and blockers before evaluating another PR/MR or dispatching more work. If nothing merges, refresh before another round.

Foundational/shared-contract implementation work must merge and appear in the refreshed state before dispatching dependents. An open parent Spec/PRD/map does not need to close first.

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

**Final-answer gate:** While workers are running and status remains observable, wait for their results and resume the loop. Normally, emit a final response only after every dispatched worker has a collected terminal status and one stop condition below is true. If status tools become unavailable and safe recovery cannot restore observation, report the loop as blocked with unresolved workers marked status unknown, their last known state, and recovery details. Unknown status is not completion: do not merge, remove the worktree, or dispatch a replacement for that ticket until the original worker's state is reconciled.

Stop under any condition below; record the blocker rather than dispatching dependent or adjacent work:

- No open `ready-for-agent` implementation ticket remains, or none can be safely dispatched.
- Foundational/shared-contract implementation work is blocked/awaiting review, or a failed merge gate blocks dependents.
- Forge, tracker, git, or subagent tools are unavailable, or failed tests/checks resist safe targeted follow-up.
- Dirty/conflicting state the loop did not create overlaps the planned work or prevents required verification or merge operations. Preserve unrelated dirty files; they alone do not block the loop.
- A new product, scope, security, or policy decision requires human input, or repo protection blocks progress. Routine confirmation of a gate-passing merge is never a human decision or stop condition.

Carry forward the dispatch ledger and add:

- Round number and draft/ready state.
- Current verification commands/results, gate result, merge SHA or reason not merged, and linked-ticket closure.
- Undispatched ready tickets grouped by reason, exact stop condition, and next-loop conditions.
