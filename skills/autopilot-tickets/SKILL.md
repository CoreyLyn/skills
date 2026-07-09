---
name: autopilot-tickets
description: Continuously drain ready-for-agent implementation tickets or issues by repeatedly dispatching safe batches to subagents, waiting for handoff artifacts, verifying merge gates, merging only safe agent-created PRs or MRs when a forge exists, refreshing ticket state, and continuing until no ready-for-agent tickets remain. Supports GitHub Issues, GitLab Issues, and local Markdown tickets under `.scratch/`, with GitHub PR, GitLab MR, or local-only handoff. Use when the user explicitly asks Codex or Claude Code to run an automatic ticket-processing loop, auto-merge completed agent PRs/MRs, or keep dispatching ready ticket work until the queue is empty.
---

# Autopilot Tickets

## Purpose

Run a controlled issue-draining loop: refresh state, process tracked handoff artifacts, merge only tracked PRs/MRs that pass every gate when a forge exists, dispatch one safe batch when nothing is mergeable or verifiable, and stop when the queue or safety conditions require it.

## Delegation

- `$dispatch-tickets` owns per-ticket readiness, branch/worktree setup, `$implement` subagent prompting, and subagent lifecycle.
- `$autopilot-tickets` owns loop state: refreshing tracker/forge data, detecting new blockers, deciding when to dispatch, converting verified drafts to ready-for-review when a forge exists, applying merge gates, handling local-only verification, and stopping.

Do not bypass `$dispatch-tickets` unless unavailable. If unavailable, apply the same readiness and lifecycle rules manually and report the fallback.

Treat `$dispatch-tickets`'s `<project-root>/.worktrees/<branch-name>` layout as a hard invariant when verifying ledgers and merge gates. Do not accept or create ticket worktrees elsewhere.

Use the repo's configured tracker and forge tools. Treat tracker and forge as separate capabilities: the tracker supplies ticket state; the forge supplies PR/MR handoff and merge automation if one exists.

## Tracker And Forge Modes

Support these tracker modes:

- **GitHub Issues**: use the repo's GitHub issue workflow.
- **GitLab Issues**: use the repo's GitLab issue workflow.
- **Local Markdown**: use tickets under `.scratch/` or the local path described in `docs/agents/issue-tracker.md`.

Support these forge modes:

- **GitHub PRs**: verify, undraft, and merge PRs through the repo's normal GitHub flow.
- **GitLab MRs**: verify, mark ready, and merge MRs through the repo's normal GitLab flow.
- **Local-only**: no PR/MR exists. Verify the committed branch and local ticket handoff, then mark the ticket `ready-for-human-merge` or the closest local equivalent. Do not auto-merge local-only branches unless the user explicitly authorizes local merges for this run.

If the tracker is local Markdown but a GitHub or GitLab remote exists, use local Markdown for ticket state and the remote forge for PR/MR handoff. If no forge exists, use local-only handoff.

## Loop

Each round:

1. Run `git status --short --branch`, fetch remotes, identify base branch, and resolve repo root for ledger verification.
2. Query tracker/forge state: open `ready-for-agent` issues at a high level, tracked handoff artifacts from prior rounds, labels or local status fields, linked issues, project fields, milestones, dependency notes, and recent comments.
3. Rebuild global dependency/blocker state for stop conditions and merge gates; leave per-ticket dispatchability to `$dispatch-tickets`.
4. Process tracked handoff artifacts before dispatching more work.
5. If no tracked handoff is mergeable or ready for local verification, call `$dispatch-tickets` for at most 3 currently safe tickets.
6. Wait for dispatch results, evaluate every returned handoff. For forge handoffs, convert verified drafts to ready-for-review, refresh PR/MR state, then merge only if all gates still pass. For local-only handoffs, verify branch, commit, diff scope, tests, and ticket handoff, then mark ready for human merge unless this run is explicitly authorized to merge local branches.
7. Confirm linked issues closed or updated.
8. Refresh issue/dependency state before the next round.

Never dispatch from a stale issue snapshot after a merge. Foundational or shared-contract work must merge and refresh before dependent issues are dispatched.

## Verification And Merge Gates

For GitHub PR and GitLab MR handoffs, merge only PRs/MRs that pass every gate:

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

For local-only handoffs, verify every gate that can exist without a forge:

- Created by this loop or a tracked prior round.
- Linked to the assigned local ticket.
- Branch and worktree match the dispatch ledger.
- Parent verified subagent `DONE` status, branch, commit, diff scope, ticket `## Handoff` note, and tests.
- Diff stays within acceptance criteria.
- No failed local checks, unresolved concerns, requested human decisions, new blocker labels/status, or dependency changes since dispatch.
- No repo rule, policy, or dirty/conflicting state blocks integration.

Local-only default outcome is `ready-for-human-merge`, not automatic merge. If the user explicitly authorized local merges for this run, merge with the repo's normal local merge method, update the local ticket to `resolved` or `merged`, and record the merge SHA. Otherwise stop after verification and report the exact branch and ticket state.

## Stop And Report

Stop when:

- No open ready-for-agent implementation issues remain.
- No currently ready issue can be safely dispatched.
- A foundational/shared-contract issue is blocked or waiting on review.
- A merge or local verification gate fails in a way that blocks dependent work.
- Required forge, tracker, git, or subagent tools are unavailable.
- Tests/checks fail and targeted follow-up cannot safely resolve them.
- The repo enters a dirty or conflicting state the loop did not create.
- User, repo policy, branch protection, or review requirements need human input.

Do not loop around blockers by dispatching dependent or adjacent work.

Track and report:

- Round number.
- Issue id/title and readiness reason.
- Branch, worktree, subagent id, final status.
- Commit SHA, handoff artifact (PR URL, MR URL, or local ticket path/section), draft/ready/local handoff state.
- Verification commands/results.
- Merge or local verification gate result, merge SHA when merged, or reason not merged.
- Linked issue closure status.
- Ready-for-agent issues not dispatched, grouped by reason.
- Exact stop condition and conditions for the next loop.

Keep the report concise but auditable.
