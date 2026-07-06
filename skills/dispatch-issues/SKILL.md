---
name: dispatch-issues
description: Analyze a repository's open issues, dependencies, labels, documentation, and working tree state, then dispatch only currently unblocked implementation issues to subagents. Use when the user asks Codex or Claude Code to triage ready issue work for AFK agents, fan out implementation tasks, dispatch subagents, start independent issue branches/worktrees, or run the next safe batch of implementation work from an issue tracker.
---

# Dispatch Issues

## Purpose

Run one safe dispatch round. Select only ready, unblocked, implementation-oriented issues, start one subagent per issue, wait for results, verify the handoff, and report the ledger.

This skill does not merge PRs/MRs or loop through the queue. For repeated dispatch, merge gates, and queue draining, use `$autopilot-issues`.

## Tooling

Use native subagent tools. Do not emulate subagents with prose or ordinary follow-up threads.

| Intent | Claude Code | Codex |
| --- | --- | --- |
| Dispatch | `Task` / `Agent` | `spawn_agent` |
| Wait | automatic return / completion notification | `wait_agent` |
| Follow up | `SendMessage` | `send_input` |
| Release slot | automatic/no-op | `close_agent` |
| Track parent work | `TodoWrite` / `TaskUpdate` | `update_plan` |

If the dispatch tool is not visible, use tool discovery for `subagent spawn agents multi-agent task`. If no native dispatch tool exists, stop and report the blocker.

Use the repo's forge/tracker tooling for PR/MR state changes. Prefer GitHub/GitLab connectors; otherwise use `gh` or `glab`.

## Intake

Before selecting issues:

1. Run `git status --short --branch`.
2. Identify base branch, current branch, remotes, tracker, and forge.
3. Read repo guidance such as `AGENTS.md`, `CLAUDE.md`, `CONTEXT.md`, `README.md`, and relevant docs.
4. Query open issues from the configured tracker.
5. Read labels, milestones, acceptance criteria, linked issues, dependency notes, and recent comments.
6. Treat dirty files that overlap expected edits as a dispatch risk unless clearly unrelated.

## Readiness

Dispatch an issue only when all are true:

- Open, implementation-oriented, and has concrete acceptance criteria or an unambiguous outcome.
- Not blocked, duplicate, stale, closed, design-only, discussion-only, `needs-info`, `needs-triage`, `ready-for-human`, or `wontfix`.
- No unresolved product/design decision, missing context, conflicting comments, or unfinished dependency.
- Work can be isolated to one branch and one worktree.
- Repo docs are sufficient for an AFK subagent.
- Work does not require secrets, privileged production access, destructive data changes, or unresolved policy choices.

If one unfinished foundational/schema/API/architecture issue blocks others, dispatch only that issue. Dispatch at most 3 subagents at once.

## Dispatch Procedure

For each selected issue:

1. Resolve repo root with `git rev-parse --show-toplevel`.
2. Create one branch and one worktree at `<project-root>/.worktrees/<branch-name>`.
3. Make the branch name include the issue id and short slug, with no path separators and safe as one Windows directory name.
4. Create `<project-root>/.worktrees` if needed; fail rather than reuse a non-empty path owned by another task.
5. Keep unrelated user changes out of the worktree.
6. Confirm the worktree starts from the correct base branch.
7. Dispatch exactly one worker/implementation subagent for exactly one issue.
8. Instruct the subagent to use `$implement`; include the `$implement` item or local `SKILL.md` path when structured tool input supports it.
9. Record agent id, issue id, branch, worktree, and expected PR/MR.

Never create issue worktrees outside `<project-root>/.worktrees/`. Never do the implementation in the parent agent unless the user explicitly asks for that fallback.

## Subagent Prompt

Give each subagent this fixed contract plus repo-specific commands, issue links, and acceptance criteria:

```text
You are assigned exactly one issue: <issue id and title>.

Repository worktree: <absolute path>
Branch: <branch>

Read the repository guide, README, relevant docs, and the assigned issue first. Implement only this issue's stated acceptance criteria. Do not implement adjacent issues or speculative improvements.

Use $implement for development: keep scope limited, add or update focused tests when behavior is testable, run relevant verification, review the work, and commit on this branch.

Before changing files, inspect the working tree and relevant code. Preserve unrelated user changes. If the issue is blocked, ambiguous, or unsafe, stop and report why instead of guessing.

After implementation, run appropriate tests, create a local commit, push the branch, and open a draft PR/MR. The PR/MR description must include the linked issue, change summary, and verification results. Leave it draft; the parent will mark it ready only after verification.

Report exactly one status:
- DONE: implementation, verification, commit, push, and draft PR/MR are complete.
- DONE_WITH_CONCERNS: work is complete, but correctness, scope, test, push, or PR/MR concerns remain.
- NEEDS_CONTEXT: specific missing information is needed.
- BLOCKED: the issue cannot be completed safely.

Always include changed files, verification commands/results, branch, commit SHA if created, PR/MR URL if opened, and remaining risks.
```

## Parent Verification

Wait for all dispatched agents before claiming the round is complete. Handle statuses mechanically:

- `DONE`: verify branch, commit, PR/MR URL, changed-file scope, tests, and PR/MR description. If clean and still draft, convert to ready-for-review with the forge tool.
- `DONE_WITH_CONCERNS`: inspect concerns; send a targeted follow-up only if the issue can still be completed safely.
- `NEEDS_CONTEXT`: provide only the missing context, or stop for human input if it cannot be discovered safely.
- `BLOCKED`: record the blocker and do not dispatch dependent work.

Do not convert PRs/MRs with failed verification, missing tests, broad scope, unresolved concerns, merge conflicts, failed checks, or new blocker labels/comments.

## Final Ledger

Report:

- Dispatched issues, readiness reason, branch, worktree, and subagent id.
- Final status, commit, PR/MR URL, draft/ready state, verification results, and remaining risks.
- Issues considered but not dispatched, grouped by reason.
- Conditions for the next round.

The round is complete only after subagent results are collected and parent verification is recorded.
