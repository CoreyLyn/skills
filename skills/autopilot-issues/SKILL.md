---
name: autopilot-issues
description: Continuously drain ready-for-agent implementation issues by repeatedly dispatching safe batches to subagents, waiting for draft PRs or MRs, verifying merge gates, merging only safe agent-created PRs or MRs, refreshing issue state, and continuing until no ready-for-agent issues remain. Use when the user explicitly asks Codex or Claude Code to run an automatic issue-processing loop, auto-merge completed agent PRs/MRs, or keep dispatching ready issue work until the queue is empty.
---

# Autopilot Issues

## Purpose

Run a controlled issue-draining loop:

1. Refresh repo, issue, dependency, label, and PR/MR state.
2. Merge any tracked PR/MR that passes all gates.
3. Use `$dispatch-issues` for one safe batch when no mergeable tracked PR/MR is waiting.
4. Wait for draft PRs/MRs, verify them, convert safe drafts to ready-for-review, then refresh state.
5. Repeat until a stop condition is reached.

If readiness, merge safety, or issue closure is uncertain, stop or leave that item unmerged.

## Delegation

Use `$dispatch-issues` as the single-round dispatcher. It owns readiness filtering, branch/worktree setup, `$implement` subagent prompting, and subagent wait/follow-up lifecycle.

Do not bypass `$dispatch-issues` unless unavailable. If unavailable, apply the same readiness and lifecycle rules manually and report the fallback.

All issue worktrees must use `<project-root>/.worktrees/<branch-name>`, with `<project-root>` from `git rev-parse --show-toplevel`. Do not accept or create issue worktrees elsewhere.

Use the repo's configured tracker and forge tools. Prefer GitHub/GitLab connectors; otherwise use `gh` or `glab`.

## Loop

Each round:

1. Run `git status --short --branch`, fetch remotes, identify base branch, and resolve repo root.
2. Query open `ready-for-agent` issues, tracked PRs/MRs from prior rounds, labels, linked issues, project fields, milestones, and recent comments.
3. Rebuild dependency/blocker state from current data.
4. Process tracked PRs/MRs before dispatching more work.
5. If no tracked PR/MR is mergeable, call `$dispatch-issues` for at most 3 currently safe issues.
6. Wait for dispatch results through `$dispatch-issues`.
7. Evaluate gates for every returned PR/MR.
8. Convert verified drafts to ready-for-review, refresh PR/MR state, then merge only if all gates still pass.
9. Confirm linked issues closed or updated.
10. Refresh issue/dependency state before the next round.

Never dispatch from a stale issue snapshot after a merge. Foundational or shared-contract work must merge and refresh before dependent issues are dispatched.

## Merge Gates

Merge only PRs/MRs that pass every gate:

- Created by this loop or a tracked prior round.
- Linked to the assigned issue.
- Source branch and worktree match the dispatch ledger.
- Parent verified subagent `DONE` status, branch, commit, diff scope, PR/MR description, and tests.
- Converted from draft to ready-for-review by the parent after verification.
- Diff stays within acceptance criteria.
- Required checks pass.
- No unresolved review comments, merge conflicts, failed checks, requested human decisions, new blocker labels/comments, or dependency changes since dispatch.
- No repo rule, branch protection, missing approval, or policy blocks the merge.

Use the repo's normal merge method. Do not invent squash/rebase/merge policy.

Do not auto-merge PRs/MRs with human-authored changes mixed in, broadened scope, production secrets, deployment controls, destructive migrations/data changes, payments, auth/access policy, or legal/compliance text unless explicitly authorized and all required reviews passed.

## Stop Conditions

Stop and report when:

- No open ready-for-agent implementation issues remain.
- No currently ready issue can be safely dispatched.
- A foundational/shared-contract issue is blocked or waiting on review.
- A merge gate fails in a way that blocks dependent work.
- Required forge, tracker, git, or subagent tools are unavailable.
- Tests/checks fail and targeted follow-up cannot safely resolve them.
- The repo enters a dirty or conflicting state the loop did not create.
- User, repo policy, branch protection, or review requirements need human input.

Do not loop around blockers by dispatching dependent or adjacent work.

## Ledger And Report

Track and report:

- Round number.
- Issue id/title and readiness reason.
- Branch, worktree, subagent id, final status.
- Commit SHA, PR/MR URL, draft/ready state.
- Verification commands/results.
- Merge gate result, merge SHA, or reason not merged.
- Linked issue closure status.
- Ready-for-agent issues not dispatched, grouped by reason.
- Exact stop condition and conditions for the next loop.

Keep the report concise but auditable.
