---
name: autopilot-spec
description: Take a spec from implementation through acceptance, review, repair, and automatic merge. Use when the user wants the whole spec delivered through one integration PR/MR using implement-spec.
---

# Autopilot Spec

Complete one spec through `implement-spec`, then verify and merge its integration PR/MR. Explicit invocation authorizes in-scope implementation, repairs, commits, pushes, and merge when all gates pass, without repeated confirmation. Honor narrower user instructions; a review-only request skips implementation and all mutations. Implicit selection grants no additional authorization. Deployment is outside scope.

## Workflow

1. **Resolve.** Read repository guidance, the spec, associated tickets, acceptance criteria, and tracker/forge configuration. Identify the target branch and any existing integration branch/PR/MR; preserve unrelated changes. Resolve missing or ambiguous inputs before implementation.
2. **Implement.** Invoke the installed `implement-spec` with those context pointers and any existing progress. If the runtime lacks a direct skill-call tool, read its `SKILL.md` and execute that workflow. If the skill or required capabilities are unavailable, stop with the missing prerequisite. Delegate ticket scheduling, worktrees, and integration to it; reuse existing work on resume. Wait for all workers and collect the integration result before continuing. If implementation is already complete, continue to acceptance; if already merged, verify closeout.
3. **Accept and review.** Reuse or create one integration PR/MR. Record head/base SHAs and a compact table covering every spec criterion, including cross-ticket behavior: evidence and `PASS`, `FAIL`, or `UNVERIFIED`. Run relevant and repository-required checks on the integration head. Review the whole diff against repository standards and the spec using `code-review`, or both axes directly if unavailable. Reuse applicable review evidence from `implement-spec`; examine subsequent changes and existing PR/MR comments. Ticket closure and worker reports alone are not acceptance evidence.
4. **Repair.** Fix in-scope defects on the integration branch, add appropriate regression coverage, commit, and update the PR/MR without overwriting concurrent work. Recheck affected behavior, required checks, and review findings. Continue while making progress; stop for repeated failures without progress, missing verification capability, or a required human decision. Unavailable or explicitly human acceptance remains `UNVERIFIED` until evidenced.
5. **Merge.** Promote a draft when acceptance and review pass. Refresh forge state and require every gate below, then automatically merge using the repository's normal method. Observe pending checks or queued merges with bounded waits; report an unresolved blocker if observation cannot continue. A queued merge is not completion.
6. **Close out.** Confirm the merge SHA and spec/ticket closure according to tracker conventions. Clean up only owned, clean worktrees whose work is integrated and workers have finished, using managed cleanup when available; never force removal. Report the PR/MR, acceptance evidence, verification results, merge SHA or precise blocker, and any remaining closeout work.

## Merge Gates

- Every spec criterion passes, including required human acceptance; the diff stays within agreed scope.
- Review findings and blocking comments are resolved; required approvals and branch protections are satisfied.
- Required local and forge checks pass, none are pending or failed, and the PR/MR is mergeable.
- The reviewed/tested head and target still match current state. Reassess and reverify affected work after head/base changes; use an expected-head merge guard when supported.
- The user's authorization covers the merge and any separately restricted changes. Preserve review-only or wait-for-approval instructions.
