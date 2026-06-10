---
name: dispatch-issues
description: Analyze a repository's open issues, dependencies, labels, documentation, and working tree state, then dispatch only currently unblocked implementation issues to subagents. Use when the user asks Codex or Claude Code to triage ready issue work for AFK agents, fan out implementation tasks, dispatch subagents, start independent issue branches/worktrees, or run the next safe batch of implementation work from an issue tracker.
---

# Dispatch Issues

## Core Rule

Dispatch only work that is clearly ready, unblocked, implementation-oriented, and safe to start now. If readiness is uncertain, do not dispatch it.

Never dispatch issues labeled or equivalent to `needs-info`, `needs-triage`, `ready-for-human`, `wontfix`, blocked, duplicate, closed, stale, design-only, discussion-only, or any task with unfinished dependencies.

This skill is an explicit request to use subagents after the readiness filter selects at least one task. Do not stop at a written dispatch plan when safe tasks were selected; actually dispatch the subagents through the available subagent tool.

This skill performs one dispatch round. It does not automatically merge PRs/MRs or continue looping through the issue queue. For an automatic loop that waits for subagent PRs/MRs, applies merge gates, merges safe work, refreshes issue state, and repeats until no ready-for-agent issues remain, use `$autopilot-issues`.

## Tool Mapping

Use the platform's native subagent tools. Do not emulate subagents with prose or ordinary follow-up threads.

This skill is written in platform-neutral terms. Map each intent to your platform's native subagent tools:

| Intent | Claude Code | Codex |
| --- | --- | --- |
| Dispatch one subagent | `Task` / `Agent` | `spawn_agent` |
| Dispatch independent subagents in parallel | Multiple `Task` calls in one message | Multiple `spawn_agent` calls before waiting |
| Wait for subagent result | Returns automatically; for background agents, resume on the completion notification | `wait_agent` |
| Send clarification or fix request | `SendMessage` to the background/named agent | `send_input` |
| Release completed subagent slot | Automatic on completion (no-op) | `close_agent` |
| Track parent checklist | `TodoWrite` (or `TaskCreate` + `TaskUpdate`) | `update_plan` |

Pick the column for the platform you are running on. The rest of this skill describes subagent actions in platform-neutral terms (dispatch, wait for result, send a follow-up, release the slot); map each to your platform's tool using the table above. Codex-only parameters are called out inline where they matter.

If your platform's subagent-dispatch tool is not visible, search for `subagent spawn agents multi-agent task` with tool discovery first. If no native subagent-dispatch tool exists on this platform even after discovery, stop and report that subagent dispatch is blocked; do not silently switch to manual implementation.

Use the repository forge or tracker tool to move PRs/MRs from draft to ready-for-review after parent verification. For GitHub, prefer the GitHub connector or `gh pr ready`. For GitLab, prefer the GitLab connector or the configured `glab` equivalent. If the ready/undraft operation is unavailable, leave the PR/MR as draft and record that blocker.

## Repository Intake

1. Inspect the current repository state before choosing tasks:
   - Run `git status --short --branch`.
   - Identify the default branch and current branch.
   - Inspect remotes and issue tracker context.
   - Read the repository guide files, such as `AGENTS.md`, `CLAUDE.md`, `CONTEXT.md`, `README.md`, and relevant docs.
2. Discover open issues from the repo's configured tracker:
   - Prefer installed GitHub or GitLab connector tools when available.
   - Use `gh`, `glab`, or local markdown issue files only when that is how the repo is managed.
   - Read labels, milestones, dependency references, linked issues, acceptance criteria, and recent comments.
3. Map issue dependencies:
   - Treat explicit dependencies, "blocked by", "depends on", parent/child sequencing, project board state, prerequisite labels, and referenced foundational tasks as blockers.
   - Treat missing acceptance criteria, ambiguous scope, or conflicting comments as uncertainty.
   - Treat dirty working tree files that would overlap an issue's expected edits as a dispatch risk unless they are clearly unrelated.

## Readiness Filter

An issue is dispatchable only when all checks pass:

- It is open and implementation-oriented.
- It has concrete acceptance criteria or an unambiguous expected outcome.
- It is not labeled or described as needing human input, triage, design decision, product decision, or more information.
- All prerequisite issues are finished or irrelevant.
- The expected work can be isolated to its own branch and worktree.
- The repo has enough documentation for an AFK subagent to proceed without guessing.
- The work does not require secrets, privileged production access, destructive data changes, or unresolved policy choices.

If a foundational infrastructure, contract, schema, migration, protocol, shared API, or cross-cutting architecture issue is unfinished and blocks multiple other issues, dispatch only that foundational issue in the current round.

Dispatch at most 3 subagents at the same time.

## Dispatch Setup

Before creating branches or prompts, load the subagent tool surface:

1. Confirm your platform's dispatch tool (see Tool Mapping) is available; if it is not visible, search for `subagent spawn agents multi-agent task` with tool discovery first.
2. Use that dispatch tool for dispatch.
3. Dispatch implementation issues as a worker/implementation subagent (on Codex, `agent_type: "worker"`).
4. Do not use separate chat threads or sessions as a substitute for subagents unless the user explicitly asks for separate threads instead of subagents.

For each selected issue:

1. Create one branch and one Git worktree dedicated to that issue.
2. Resolve the current project's root with `git rev-parse --show-toplevel`.
3. Use a branch name that includes the issue identifier and a short slug, and is safe to use as one Windows directory name without path separators.
4. Create the worktree only at `<project-root>/.worktrees/<branch-name>`. Do not create issue worktrees as siblings of the project, in a global worktree directory, or anywhere else.
5. Create `<project-root>/.worktrees` when it does not exist, and fail rather than reuse a non-empty path that belongs to another branch or task.
6. Keep existing user changes out of the new worktree unless they are intentionally part of the issue.
7. Confirm the worktree starts from the correct base branch.
8. Start exactly one subagent for exactly one issue.
9. Instruct the subagent to prefer `$tdd` for development when the issue is implementation work with testable behavior.
10. When the dispatch tool supports structured items, include the `$tdd` skill item or its local `SKILL.md` path in the subagent input when available.

For example, branch `issue-42-add-json-report` must use:

```text
<project-root>/.worktrees/issue-42-add-json-report
```

Treat the resolved project root as authoritative even when the skill is invoked from a subdirectory.

Do not simulate dispatch by doing the implementation in the parent agent unless the user explicitly asks for that fallback.

## Dispatch Lifecycle

Use this exact lifecycle after selecting issues:

1. Create or verify the dedicated branch and worktree for each selected issue at `<project-root>/.worktrees/<branch-name>`.
2. Build a complete subagent prompt for each issue.
3. Dispatch one subagent per issue (one dispatch call each), as a worker/implementation subagent for implementation tasks (on Codex, `agent_type: "worker"`).
4. Record each returned agent id, issue id, branch, and worktree path.
5. After all selected subagents are dispatched, wait for the active subagents' results.
6. If a subagent returns `NEEDS_CONTEXT`, answer with a clarification/fix request and wait again.
7. If a subagent returns `DONE_WITH_CONCERNS`, inspect the concerns before accepting completion.
8. If a subagent returns `BLOCKED`, do not retry blindly; decide whether more context, a smaller task, or human input is required.
9. If a subagent returns `DONE`, verify the reported branch, commit, PR/MR URL, changed-file scope, and test results.
10. If parent verification passes and the PR/MR is still draft, convert it to ready-for-review with the repository forge tool. Do not convert PRs/MRs with concerns, failed verification, missing tests, unclear scope, or unresolved blockers.
11. When a subagent reaches a final status and no more follow-up is needed, release its slot.
12. Produce the dispatch ledger and next-round gate conditions.

Wait for dispatched agents before claiming the round has completed. It is acceptable to do non-overlapping parent work while agents run, but do not lose the agent ids or skip result collection.

## Subagent Prompt

Give each subagent a prompt with these constraints:

```text
You are assigned exactly one issue: <issue id and title>.

Repository worktree: <absolute path>
Branch: <branch>

First read the repository guide, README, relevant docs, and the assigned issue. Implement only this issue's stated acceptance criteria. Do not casually implement follow-up tasks, adjacent issues, or speculative improvements.

Prefer using $tdd for development: write or update focused failing tests first, make them pass with the smallest implementation, then refactor only within the assigned issue scope. If TDD is not practical for this issue, state why before implementing.

Before changing files, inspect the working tree and relevant code. Preserve unrelated user changes. If the issue is blocked, ambiguous, or unsafe, stop and report why instead of guessing.

After implementation, run the appropriate tests for the touched area. Create a local commit, push the branch, and open a draft PR/MR. Leave the PR/MR as draft; the parent agent will convert it to ready-for-review only after parent verification passes. The PR/MR description must include:
- Linked issue
- Change summary
- Verification results

When finished, report exactly one status:
- DONE: implementation, verification, commit, push, and draft PR/MR are complete.
- DONE_WITH_CONCERNS: work is complete, but there are correctness, scope, test, push, or PR/MR concerns the parent must inspect.
- NEEDS_CONTEXT: you need specific missing information before continuing.
- BLOCKED: you cannot safely complete the issue.

For any status, include changed files, verification commands/results, branch, commit SHA if created, PR/MR URL if opened, and any remaining risks.
```

Add repository-specific commands, tracker links, and acceptance criteria after the fixed constraints.

## Handling Subagent Results

Handle subagent statuses mechanically:

- `DONE`: verify the reported branch, commit, PR/MR URL, changed-file scope, and tests. If verification passes and the PR/MR is draft, convert it to ready-for-review; then release the subagent slot.
- `DONE_WITH_CONCERNS`: read the concerns. If they affect acceptance criteria, tests, push, or PR/MR creation, send a targeted follow-up or mark the issue as not fully dispatched-complete.
- `NEEDS_CONTEXT`: provide only the missing context or stop for human input if the context cannot be discovered safely.
- `BLOCKED`: record the blocker and do not dispatch another issue that depends on it.

Never treat a subagent success report as proof by itself. The parent must inspect the final summary, changed-file scope, and verification result before reporting the round outcome.

## Draft PR/MR Readiness

Draft PRs/MRs are the safe handoff state from subagents. The parent agent owns the transition to ready-for-review.

Convert a draft PR/MR to ready-for-review only when:

- The subagent status is `DONE`.
- The parent verified the PR/MR URL, branch, commit, changed-file scope, and tests.
- The diff stays within the assigned issue's acceptance criteria.
- The PR/MR description includes the linked issue, change summary, and verification results.
- There are no unresolved concerns, failed checks, merge conflicts, or new blocker labels/comments.

If any condition fails, leave the PR/MR as draft and record the reason.

## Parent Agent Responsibilities

After dispatching:

- Track which issue, branch, worktree, and subagent correspond to each task.
- Summarize why each dispatched issue was ready.
- Summarize each subagent's final status, commit, PR/MR URL, draft/ready state, and verification result.
- Summarize every issue considered but not dispatched, grouped by reason such as blocked dependency, needs info, human decision required, foundational issue not finished, unclear acceptance criteria, worktree conflict, or concurrency limit.
- State the conditions for the next dispatch round, including which PRs/MRs must merge, which blocker must resolve, or what human input is needed.

Do not mark the overall dispatch round complete merely because subagents were started. The parent's deliverable is a clear dispatch ledger and next-round gate conditions.
