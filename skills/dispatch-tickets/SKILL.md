---
name: dispatch-tickets
description: Analyze open tickets/issues, dependencies, docs, and worktree state; dispatch only currently unblocked implementation tickets to subagents. Use when the user asks to triage ready ticket work for AFK agents, fan out implementation, start independent ticket branches/worktrees, or run the next safe batch from a tracker.
---

# Dispatch Tickets

## Contract

Run one safe round: classify every candidate, dispatch every safe implementation ticket, collect all results, verify handoffs, and report a ledger. Do not merge PRs/MRs, move drafts to ready, or loop; use the `autopilot-tickets` skill for those responsibilities.

Set no skill-level concurrency cap; native capacity limits simultaneous calls only.

## Select Issues

Use the configured tracker and forge.

1. Run `git status --short --branch`; resolve the repo root; identify base/current branches, remotes, tracker, and forge.
2. Read applicable `AGENTS.md` and `CLAUDE.md` guidance; consult context, README, and other docs only as needed to assess the selected scope.
3. Query open candidates within the requested scope (default: `ready-for-agent`); read their labels, criteria, dependencies, and recent comments. Follow linked dependencies and blockers as needed to establish readiness.
4. Classify every implementation candidate in this round as selected or unselected with a reason. A parent Spec/PRD/map link supplies scope only: its open state is not a dependency. Treat only unfinished implementation-level dependencies and explicit decision/review/policy gates as blockers; treat dirty files that overlap expected edits as a dispatch risk unless clearly unrelated.

Dispatch an issue only when all are true:

- Open implementation work with concrete acceptance criteria or an unambiguous outcome.
- Not blocked, duplicate, stale, closed, design-only, discussion-only, `needs-info`, `needs-triage`, `ready-for-human`, or `wontfix`.
- No unresolved product/design decision, context gap, conflicting comments, or unfinished dependency.
- Isolatable to one branch/worktree.
- Repo docs suffice for AFK execution.
- Requires no secrets, privileged production access, destructive data changes, or unresolved policy choices.

When unfinished foundational/schema/API/architecture implementation work blocks others, dispatch only it. Do not select a design-only parent merely because it is open.

## Dispatch

**Action:** dispatch one subagent per selected issue. Prose replies and ordinary chat threads are not dispatch.

Use this platform-neutral form (map to the harness's real tool — Task/Agent, `spawn_agent`, `task`, `invoke_agent`, etc.):

```text
Subagent (general-purpose):
  description: "Implement <issue-id>: <short title>"
  prompt: |
    <Worker Contract below, filled in>
```

Rules:

- One subagent per issue; independent issues may be dispatched in the same turn (parallel).
- Do not invent tool names. If the harness has no multi-agent/subagent capability, stop and report that instead of faking dispatch.
- Prefer an explicit model only when the harness supports it; otherwise omit.
- Record agent id (if returned), issue id, branch, worktree, and expected draft PR/MR.
- Do not instruct workers to push or open a PR/MR. Parent does those after verifying the local commit.

For each selected issue:

1. Resolve `<project-root>` with `git rev-parse --show-toplevel`; choose an issue-id/slug branch with no path separators and a Windows-safe name.
2. Create its worktree only at `<project-root>/.claude/worktrees/<branch-name>`; create `.claude/worktrees` if needed and never reuse a non-empty path owned by another task.
3. Confirm the worktree starts from the correct base branch and contains no unrelated user changes.
4. Dispatch one worker per issue with the filled Worker Contract.

During autopilot, the parent may repair loop-created work within the same ticket scope when a worker or targeted follow-up fails.
Otherwise, parent implementation requires an explicit user fallback request.

## Worker Contract

Add repo commands, issue links, and acceptance criteria to this fixed contract:

```text
You are assigned exactly one issue: <issue id and title>.
Worktree: <absolute path>; branch: <branch>.
Read the guidance and issue; inspect the tree and code. Change only its acceptance scope and preserve unrelated work. Report NEEDS_CONTEXT or BLOCKED instead of guessing when ambiguous, blocked, or unsafe.
Implement the assigned issue. Run checks relevant to the change and all repo-required checks; run the full suite only when required or warranted by the impact. Invoke the /tdd skill for test-driven development or the /code-review skill for independent review when requested or justified by the task. Review the diff, then commit to the current branch.
Do not push or open a PR/MR. Stop after the local commit.
Report exactly one status: DONE (complete), DONE_WITH_CONCERNS (complete with risks), NEEDS_CONTEXT (specific information required), or BLOCKED (cannot complete safely).
Include changed files, verification commands/results, branch, commit SHA if created, and remaining risks.
```

## Verify And Report

Complete the round only after collecting every agent result and recording parent verification.

Handle statuses mechanically: `DONE`—verify branch, commit, scope, and tests, then parent pushes and opens a draft PR/MR without moving it to ready; `DONE_WITH_CONCERNS`—inspect concerns and send a targeted follow-up only if the issue can still be completed safely; `NEEDS_CONTEXT`—supply only discoverable missing context or seek human input; `BLOCKED`—record it and withhold dependents.
Verify a draft only when tests and verification pass, scope and required fields match, and no unresolved concerns, merge conflicts, failed checks, or new blocker labels/comments remain.

Ledger: each selected issue's id/title, readiness reason, branch/worktree/agent, status, commit, draft URL and verified/unverified state, verification, and risks; unselected candidates grouped by reason; exact next-round conditions.
