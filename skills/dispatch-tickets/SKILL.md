---
name: dispatch-tickets
description: Analyze a repository's open tickets or issues, dependencies, labels, documentation, and working tree state, then dispatch only currently unblocked implementation tickets to subagents. Use when the user asks Codex or Claude Code to triage ready ticket work for AFK agents, fan out implementation tasks, dispatch subagents, start independent ticket branches/worktrees, or run the next safe batch of implementation work from a tracker.
---

# Dispatch Tickets

## Contract

Run one safe round: classify every candidate, dispatch every safe implementation ticket, collect all results, verify handoffs, and report a ledger. Do not merge PRs/MRs, move drafts to ready, or loop; use the `autopilot-tickets` skill for those responsibilities.

Set no skill-level concurrency cap; native capacity limits simultaneous calls only.

## Select Issues

Use native subagent tools; prose and ordinary threads are not dispatch. Discover hidden tools with `subagent spawn agents multi-agent task` and stop if none exists. Use the configured tracker and forge.

1. Run `git status --short --branch`; resolve the repo root; identify base/current branches, remotes, tracker, and forge.
2. Read `AGENTS.md`, `CLAUDE.md`, `CONTEXT.md`, `README.md`, and relevant docs.
3. Query every open ticket; read labels, milestones, criteria, links, dependencies, and recent comments.
4. Classify every open implementation candidate as selected or unselected with a reason; treat dirty files that overlap expected edits as a dispatch risk unless clearly unrelated.

Dispatch an issue only when all are true:

- Open implementation work with concrete acceptance criteria or an unambiguous outcome.
- Not blocked, duplicate, stale, closed, design-only, discussion-only, `needs-info`, `needs-triage`, `ready-for-human`, or `wontfix`.
- No unresolved product/design decision, context gap, conflicting comments, or unfinished dependency.
- Isolatable to one branch/worktree.
- Repo docs suffice for AFK execution.
- Requires no secrets, privileged production access, destructive data changes, or unresolved policy choices.

When unfinished foundational/schema/API/architecture work blocks others, dispatch only it.

## Dispatch

For each selected issue:

1. Resolve `<project-root>` with `git rev-parse --show-toplevel`; choose an issue-id/slug branch with no path separators and a Windows-safe name.
2. Create its worktree only at `<project-root>/.worktrees/<branch-name>`; create `.worktrees` if needed and never reuse a non-empty path owned by another task.
3. Confirm the worktree starts from the correct base branch and contains no unrelated user changes.
4. Dispatch one worker per issue with the `implement` skill (skill item or local path when supported); record agent id, issue id, branch, worktree, and expected PR/MR.

During autopilot, the parent may repair loop-created work within the same ticket scope when a worker or targeted follow-up fails.
Otherwise, parent implementation requires an explicit user fallback request.

## Worker Contract

Add repo commands, issue links, and acceptance criteria to this fixed contract:

```text
You are assigned exactly one issue: <issue id and title>.
Worktree: <absolute path>; branch: <branch>.
Read the guidance and issue; inspect the tree and code. Change only its acceptance scope and preserve unrelated work. Report NEEDS_CONTEXT or BLOCKED instead of guessing when ambiguous, blocked, or unsafe.
Use the implement skill for implementation, focused tests where practical, verification, review, and commit.
After the implement skill completes and the work is committed, push the branch and open a draft PR/MR with linked issue, change summary, and verification results. Keep it draft for parent verification.
Report exactly one status: DONE (complete), DONE_WITH_CONCERNS (complete with risks), NEEDS_CONTEXT (specific information required), or BLOCKED (cannot complete safely).
Include changed files, verification commands/results, branch, commit SHA if created, draft PR/MR URL if opened, and remaining risks.
```

## Verify And Report

Complete the round only after collecting every agent result and recording parent verification.

Handle statuses mechanically: `DONE`—verify branch, commit, draft URL, scope, tests, and description without moving it to ready; `DONE_WITH_CONCERNS`—inspect concerns and send a targeted follow-up only if the issue can still be completed safely; `NEEDS_CONTEXT`—supply only discoverable missing context or seek human input; `BLOCKED`—record it and withhold dependents.
Verify a draft only when tests and verification pass, scope and required fields match, and no unresolved concerns, merge conflicts, failed checks, or new blocker labels/comments remain.

Ledger: each selected issue's id/title, readiness reason, branch/worktree/agent, status, commit, draft URL and verified/unverified state, verification, and risks; unselected candidates grouped by reason; exact next-round conditions.
