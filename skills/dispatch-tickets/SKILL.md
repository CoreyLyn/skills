---
name: dispatch-tickets
description: Analyze a repository's open tickets or issues, dependencies, labels, documentation, and working tree state, then dispatch only currently unblocked implementation tickets to subagents. Supports GitHub Issues, GitLab Issues, and local Markdown tickets under `.scratch/`, with GitHub PR, GitLab MR, or local-only handoff. Use when the user asks Codex or Claude Code to triage ready ticket work for AFK agents, fan out implementation tasks, dispatch subagents, start independent ticket branches/worktrees, or run the next safe batch of implementation work from a tracker.
---

# Dispatch Tickets

## Purpose

Run one safe dispatch round. Select only ready, unblocked, implementation-oriented issues, start one subagent per issue, wait for results, verify the handoff, and report the ledger.

This skill does not merge PRs/MRs or loop through the queue. For repeated dispatch, merge gates, and queue draining, use `$autopilot-tickets`.

## Tooling

Use native subagent tools; do not emulate subagents with prose or ordinary follow-up threads. In Codex use `spawn_agent`, `wait_agent`, `send_input`, `close_agent`, and `update_plan`; in Claude Code use `Task`/`Agent`, `SendMessage`, and normal task tracking.

If the dispatch tool is not visible, use tool discovery for `subagent spawn agents multi-agent task`. If no native dispatch tool exists, stop and report the blocker.

Use the repo's configured tracker and forge tooling. Treat them as separate capabilities: the tracker supplies ticket state; the forge supplies PR/MR handoff if one exists.

## Tracker And Forge Modes

Support these tracker modes:

- **GitHub Issues**: use the repo's GitHub issue workflow.
- **GitLab Issues**: use the repo's GitLab issue workflow.
- **Local Markdown**: use tickets under `.scratch/` or the local path described in `docs/agents/issue-tracker.md`.

Support these forge modes:

- **GitHub PRs**: push the branch and open a draft PR.
- **GitLab MRs**: push the branch and open a draft MR.
- **Local-only**: no PR/MR exists. Keep the committed branch local, append a handoff note to the ticket, and mark the ticket ready for parent verification. Do not pretend a PR/MR exists.

If the tracker is local Markdown but a GitHub or GitLab remote exists, use local Markdown for ticket state and the remote forge for PR/MR handoff. If no forge exists, use local-only handoff.

## Select Issues

1. Run `git status --short --branch`.
2. Identify base branch, current branch, remotes, tracker, and forge.
3. Read repo guidance such as `AGENTS.md`, `CLAUDE.md`, `CONTEXT.md`, `README.md`, and relevant docs.
4. Query open issues from the configured tracker.
5. Read labels, milestones, acceptance criteria, linked issues, dependency notes, and recent comments.
6. Treat dirty files that overlap expected edits as a dispatch risk unless clearly unrelated.

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
9. Record agent id, issue id, branch, worktree, tracker mode, forge mode, and expected handoff artifact.

Never create issue worktrees outside `<project-root>/.worktrees/`, reuse another task's worktree, or implement in the parent agent unless the user explicitly asks for that fallback.

## Subagent Prompt

Give each subagent this fixed contract plus repo-specific commands, issue links, and acceptance criteria:

```text
You are assigned exactly one issue: <issue id and title>.

Repository worktree: <absolute path>
Branch: <branch>

Read the repository guide, README, relevant docs, and the assigned issue first. Implement only this issue's stated acceptance criteria. Do not implement adjacent issues or speculative improvements.

Use $implement for development; it owns implementation, focused tests where practical, verification, review, and commit on this branch.

Before changing files, inspect the working tree and relevant code. Preserve unrelated user changes. If the issue is blocked, ambiguous, or unsafe, stop and report why instead of guessing.

After $implement completes and the work is committed, use the repo's configured handoff:

- GitHub/GitLab forge: push the branch and open a draft PR/MR. The PR/MR description must include the linked issue, change summary, and verification results. Leave it draft; the parent will verify and record the handoff.
- Local-only: keep the committed branch local, append a `## Handoff` note to the local ticket with the change summary and verification results, and set the ticket state to ready for parent verification using the repo's local ticket convention.

Report exactly one status: DONE, DONE_WITH_CONCERNS, NEEDS_CONTEXT, or BLOCKED.

Always include changed files, verification commands/results, branch, commit SHA if created, handoff artifact (PR URL, MR URL, or local ticket path/section), and remaining risks.
```

## Verify And Report

Wait for all dispatched agents before claiming the round is complete. Handle statuses mechanically:

- `DONE`: verify branch, commit, changed-file scope, tests, and handoff artifact. For a forge handoff, verify the draft PR/MR URL and description. For local-only, verify the local ticket `## Handoff` section and ready-for-parent-verification state. Record whether the handoff is verified; do not convert it to ready-for-review.
- `DONE_WITH_CONCERNS`: inspect concerns; send a targeted follow-up only if the issue can still be completed safely.
- `NEEDS_CONTEXT`: provide only the missing context, or stop for human input if it cannot be discovered safely.
- `BLOCKED`: record the blocker and do not dispatch dependent work.

Do not mark handoffs as verified when they have failed verification, missing tests, broad scope, unresolved concerns, merge conflicts, failed checks where checks exist, missing local handoff notes, or new blocker labels/comments.

Report:

- Dispatched issues, readiness reason, branch, worktree, and subagent id.
- Final status, commit, handoff artifact, verified/unverified handoff state, verification results, and remaining risks.
- Issues considered but not dispatched, grouped by reason.
- Conditions for the next round.

The round is complete only after subagent results are collected and parent verification is recorded.
