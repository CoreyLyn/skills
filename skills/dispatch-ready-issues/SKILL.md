---
name: dispatch-ready-issues
description: Analyze a repository's open issues, dependencies, labels, documentation, and working tree state, then dispatch only currently unblocked implementation issues to subagents. Use when the user asks Codex to triage ready issue work for AFK agents, fan out implementation tasks, dispatch subagents, start independent issue branches/worktrees, or run the next safe batch of implementation work from an issue tracker.
---

# Dispatch Ready Issues

## Core Rule

Dispatch only work that is clearly ready, unblocked, implementation-oriented, and safe to start now. If readiness is uncertain, do not dispatch it.

Never dispatch issues labeled or equivalent to `needs-info`, `needs-triage`, `ready-for-human`, `wontfix`, blocked, duplicate, closed, stale, design-only, discussion-only, or any task with unfinished dependencies.

This skill is an explicit request to use subagents after the readiness filter selects at least one task. Do not stop at a written dispatch plan when safe tasks were selected; actually spawn the subagents through the available multi-agent tool.

This skill performs one dispatch round. It does not automatically merge PRs/MRs or continue looping through the issue queue. For an automatic loop that waits for subagent PRs/MRs, applies merge gates, merges safe work, refreshes issue state, and repeats until no ready-for-agent issues remain, use `$autopilot-ready-issues`.

## Tool Mapping

Use the platform's native subagent tools. Do not emulate subagents with prose or ordinary follow-up threads.

For Codex:

| Intent | Codex tool |
| --- | --- |
| Dispatch one subagent | `spawn_agent` |
| Dispatch independent subagents in parallel | Multiple `spawn_agent` calls before waiting |
| Wait for subagent result | `wait_agent` |
| Send clarification or fix request | `send_input` |
| Release completed subagent slot | `close_agent` |
| Track parent checklist | `update_plan` |

If `spawn_agent`, `wait_agent`, or `close_agent` are not visible, search for `subagent spawn agents multi-agent` with tool discovery first. If the tools remain unavailable after discovery, stop and report that subagent dispatch is blocked; do not silently switch to manual implementation.

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

1. Search for `subagent spawn agents multi-agent` with tool discovery when `spawn_agent` is not already visible.
2. Use the discovered `spawn_agent` tool for dispatch.
3. Prefer `agent_type: "worker"` for implementation issues.
4. Do not use separate Codex threads as a substitute for subagents unless the user explicitly asks for separate threads instead of subagents.

For each selected issue:

1. Create one branch and one Git worktree dedicated to that issue.
2. Use a branch name that includes the issue identifier and a short slug.
3. Keep existing user changes out of the new worktree unless they are intentionally part of the issue.
4. Confirm the worktree starts from the correct base branch.
5. Start exactly one subagent for exactly one issue.
6. Instruct the subagent to prefer `$tdd` for development when the issue is implementation work with testable behavior.
7. When `spawn_agent` supports structured items, include the `$tdd` skill item or its local `SKILL.md` path in the subagent input when available.

Do not simulate dispatch by doing the implementation in the parent agent unless the user explicitly asks for that fallback.

## Dispatch Lifecycle

Use this exact lifecycle after selecting issues:

1. Create or verify the dedicated branch and worktree for each selected issue.
2. Build a complete subagent prompt for each issue.
3. Call `spawn_agent` once per issue, with `agent_type: "worker"` for implementation tasks.
4. Record each returned agent id, issue id, branch, and worktree path.
5. After all selected subagents are spawned, call `wait_agent` for the active agent ids.
6. If a subagent returns `NEEDS_CONTEXT`, answer with `send_input` and wait again.
7. If a subagent returns `DONE_WITH_CONCERNS`, inspect the concerns before accepting completion.
8. If a subagent returns `BLOCKED`, do not retry blindly; decide whether more context, a smaller task, or human input is required.
9. When a subagent reaches a final status and no more follow-up is needed, call `close_agent` to free the slot.
10. Produce the dispatch ledger and next-round gate conditions.

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

After implementation, run the appropriate tests for the touched area. Create a local commit, push the branch, and open a draft PR/MR. The PR/MR description must include:
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

- `DONE`: verify the reported branch, commit, PR/MR URL, and tests; then close the agent.
- `DONE_WITH_CONCERNS`: read the concerns. If they affect acceptance criteria, tests, push, or PR/MR creation, send a targeted follow-up or mark the issue as not fully dispatched-complete.
- `NEEDS_CONTEXT`: provide only the missing context or stop for human input if the context cannot be discovered safely.
- `BLOCKED`: record the blocker and do not dispatch another issue that depends on it.

Never treat a subagent success report as proof by itself. The parent must inspect the final summary, changed-file scope, and verification result before reporting the round outcome.

## Parent Agent Responsibilities

After dispatching:

- Track which issue, branch, worktree, and subagent correspond to each task.
- Summarize why each dispatched issue was ready.
- Summarize each subagent's final status, commit, PR/MR URL, and verification result.
- Summarize every issue considered but not dispatched, grouped by reason such as blocked dependency, needs info, human decision required, foundational issue not finished, unclear acceptance criteria, worktree conflict, or concurrency limit.
- State the conditions for the next dispatch round, including which PRs/MRs must merge, which blocker must resolve, or what human input is needed.

Do not mark the overall dispatch round complete merely because subagents were started. The parent's deliverable is a clear dispatch ledger and next-round gate conditions.
