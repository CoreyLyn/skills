---
name: autopilot-issues
description: Continuously drain ready-for-agent implementation issues by repeatedly dispatching safe batches to subagents, waiting for draft PRs or MRs, verifying merge gates, merging only safe agent-created PRs or MRs, refreshing issue state, and continuing until no ready-for-agent issues remain. Use when the user explicitly asks Codex or Claude Code to run an automatic issue-processing loop, auto-merge completed agent PRs/MRs, or keep dispatching ready issue work until the queue is empty.
---

# Autopilot Issues

## Core Rule

Run a controlled loop, not a one-shot dispatch. Each loop iteration must:

1. Refresh repository, issue, dependency, label, and PR/MR state.
2. Dispatch only currently safe ready-for-agent issues.
3. Wait for subagents to finish and open draft PRs/MRs.
4. Verify strict merge gates.
5. Convert verified draft PRs/MRs to ready-for-review.
6. Merge only ready PRs/MRs that pass every gate.
7. Refresh state before dispatching the next round.

If readiness, merge safety, or issue closure is uncertain, stop or leave that item unmerged. Do not guess.

## Required Sub-Skill

Use `$dispatch-issues` as the single-round dispatcher. This skill owns the outer loop, merge gates, state refresh, and stop conditions. `$dispatch-issues` owns issue readiness filtering, branch/worktree setup, subagent prompting, and subagent wait/close lifecycle.

Implementation subagents should receive `$implement` through `$dispatch-issues`; this skill should not independently prescribe a lower-level development workflow.

Do not bypass `$dispatch-issues` unless it is unavailable; if unavailable, apply its same readiness and subagent lifecycle rules manually and report that fallback.

All issue worktrees must be located at `<project-root>/.worktrees/<branch-name>`, where `<project-root>` is resolved with `git rev-parse --show-toplevel` and `<branch-name>` is safe to use as one Windows directory name without path separators. Do not accept or create worktrees outside this directory layout.

## Tool Requirements

Use the repository's configured tracker and forge tools:

- GitHub: prefer GitHub connector tools when available; otherwise use `gh`.
- GitLab: prefer GitLab connector tools when available; otherwise use `glab`.
- Local markdown issues: use local files only when the repo manages issues that way.

Use the platform's native subagent tools through `$dispatch-issues` (see its Tool Mapping for full Claude Code / Codex equivalents):

| Intent | Claude Code | Codex |
| --- | --- | --- |
| Dispatch subagent | `Task` / `Agent` | `spawn_agent` |
| Wait for subagent | Returns automatically; for background agents, resume on the completion notification | `wait_agent` |
| Follow up with subagent | `SendMessage` to the background/named agent | `send_input` |
| Close completed subagent | Automatic on completion (no-op) | `close_agent` |
| Track loop status | `TodoWrite` (or `TaskCreate` + `TaskUpdate`) | `update_plan` |

## Loop Algorithm

Repeat until a stop condition is reached:

1. Refresh base state:
   - Run `git status --short --branch`.
   - Fetch remotes.
   - Identify the base branch.
   - Resolve the project root and verify tracked issue worktrees use `<project-root>/.worktrees/<branch-name>`.
   - Query open issues labeled or equivalent to `ready-for-agent`.
   - Query open PRs/MRs created by prior loop rounds.
   - Rebuild dependency and blocker state from current labels, linked issues, project fields, milestones, and comments.
2. If there are mergeable PRs/MRs from prior rounds, process merge gates before dispatching more work.
3. If no mergeable PRs/MRs exist, call `$dispatch-issues` for one safe batch of at most 3 issues.
4. Wait for the dispatched subagents to complete through `$dispatch-issues`.
5. For each returned PR/MR, evaluate merge gates.
6. For each draft PR/MR that passes parent verification and merge gates except draft state, convert it to ready-for-review with the repository forge tool.
7. Re-check required checks, comments, conflicts, labels, and dependencies after conversion.
8. Merge PRs/MRs that are ready-for-review and pass every gate.
9. Confirm each merged PR/MR closed or updated its linked issue. If the issue remains open, refresh and determine whether more ready work remains.
10. Re-query all issue and dependency state before the next dispatch round.

Never dispatch from a stale issue snapshot after a merge. Foundational or shared-contract changes must be merged and state-refreshed before dependent issues are dispatched.

## Merge Gates

Merge only PRs/MRs that pass all gates:

- The PR/MR was created by a subagent in this loop or by a tracked prior round.
- The PR/MR is linked to exactly the assigned issue unless the tracker explicitly permits multiple linked issues and all are ready.
- The source branch and worktree match the dispatch ledger.
- The PR/MR has been converted from draft to ready-for-review by the parent after verification.
- The diff is limited to the assigned issue's acceptance criteria.
- The PR/MR description includes linked issue, change summary, and verification results.
- Required checks are successful.
- The subagent ran appropriate tests and reported results.
- The parent agent has inspected the subagent final status and changed-file scope.
- There are no unresolved review comments, failed checks, merge conflicts, or requested human decisions.
- The issue has no new blocker, needs-info comment, changed label, or dependency update since dispatch.
- The PR/MR is not blocked by repository rules, branch protection, missing approvals, or policy.

If any gate fails, do not merge. Record the reason and either send a targeted subagent follow-up, leave the PR/MR for human review, or stop the loop if the blocker affects further dispatch.

## Draft to Ready

Subagents create draft PRs/MRs as the handoff state. The parent agent owns the transition to ready-for-review.

Convert a draft PR/MR to ready-for-review only when:

- The subagent returned `DONE`.
- Parent verification confirms branch, commit, changed-file scope, PR/MR description, and test results.
- The issue still has no new blocker, dependency change, label change, or needs-info comment.
- The diff is limited to acceptance criteria.
- No repository policy requires human review before undrafting.

After conversion, refresh PR/MR state before merging. Some checks, automations, or review rules may run only after a PR/MR becomes ready-for-review.

If conversion fails or the forge tool cannot undraft the PR/MR, leave it as draft, record the blocker, and do not merge it.

## Auto-Merge Boundaries

Automatic merge is allowed only for PRs/MRs that pass every merge gate. Do not auto-merge:

- PRs/MRs not created by this loop or a tracked prior round.
- PRs/MRs with human-authored changes mixed into the branch.
- PRs/MRs touching production secrets, deployment controls, migrations, destructive data changes, payments, authentication policy, access control policy, or legal/compliance text unless the issue explicitly authorized this and all required reviews passed.
- PRs/MRs that broaden scope beyond acceptance criteria.
- PRs/MRs where the issue, labels, dependencies, or base branch changed after dispatch.

Use the repository's normal merge method. Prefer squash/rebase/merge according to repo policy, branch protection, or documented conventions; do not invent a merge style.

## Stop Conditions

Stop the loop and report when any condition occurs:

- No open ready-for-agent issues remain.
- A foundational issue or shared contract issue is blocked or waiting on review.
- No currently ready issue can be safely dispatched.
- A merge gate fails in a way that blocks dependent work.
- Required forge, tracker, git, or subagent tools are unavailable.
- Tests or checks fail and a targeted follow-up cannot safely resolve them.
- The repository enters a dirty or conflicting state that the loop did not create.
- The user, repository policy, branch protection, or review requirement requires human input.

Do not continue looping around a blocker by dispatching dependent or adjacent work.

## Dispatch Ledger

Maintain a ledger during the loop:

- Round number.
- Issue id and title.
- Reason it was ready.
- Branch and worktree.
- Subagent id.
- Subagent final status.
- Commit SHA.
- PR/MR URL.
- Draft/ready state.
- Verification commands and results.
- Merge gate result.
- Merge SHA or reason not merged.
- Linked issue closure status.

Use the ledger for the final summary and for deciding whether a PR/MR belongs to the loop.

## Final Report

When the loop stops, report:

- Issues dispatched and merged.
- Issues dispatched but not merged, with gate failure or blocker.
- Ready-for-agent issues not dispatched, grouped by reason.
- PRs/MRs still open.
- Draft PRs/MRs that could not be converted to ready-for-review.
- Verification summary.
- Exact stop condition.
- Conditions required for the next loop.

Keep the report concise but complete enough that a human can audit what was merged and why.
