---
name: phase-flow
description: Run a phased plan of Jira stories end to end, unattended. For each phase, stack a temp branch with every earlier phase's PRs merged, then build each story of the phase one at a time in subagents (split, seams, TDD, draft PR with the plan's labels), run the review loop, get CI green and hand the story to review. Never asks the user anything.
argument-hint: "<plan-file | epic-key> [plan] [from <phase>]"
disable-model-invocation: true
---

# Phase flow

Target: `$ARGUMENTS`. The **plan** is a markdown file, or an epic key whose description lists the phases. `from <n>` starts at phase n; without it the run resumes from its state file. `plan` stops after preflight: the plan is written, checked and seeded, nothing is built, and the phase table is printed for the user to review before the real run.

This is ticket-flow run unattended across a whole plan. Every place where ticket-flow, pr-loop or tdd would ask the user, phase-flow decides, writes the decision down, and keeps going.

**The main context only orchestrates.** It holds keys, branch names, PR numbers and one-line outcomes. Story bodies, code and logs stay inside subagents. The review loop is the one exception: it runs here, because pr-review spawns its own subagents and subagents can't nest. That fills the context, so expect compactions. **The state file is the memory**: after a compaction or a restart, re-read it and continue from the first row that is neither `ready` nor `blocked`. The subagent briefs live in [briefs.md](briefs.md); read it at the start and again after a compaction.

Jira goes through the Atlassian MCP tools (cloudId from `getAccessibleAtlassianResources`). GitHub goes through `gh`.

## Guardrails

- The run pushes to its own branches (story branches and `tmp-<plan>-phase-<n>`) and to the head branches of the plan's open PRs. `main` is only ever read. Merging PRs is the user's job.
- Make CI green by fixing code. Tests, coverage thresholds, lint config and workflows change only when the story is about them, and hooks always run.
- Open every PR as a draft, with the plan's labels if it sets any.
- Jira changes are limited to the plan's stories: assign to the user, move to In Progress, comment the PR link, move to review.
- When progress needs a human (credentials, an external system, a decision the story, the epic and the code can't settle), mark the slice `blocked` with the reason and move to the next story. When Jira or `gh` authentication fails, stop the run and report.

## Plan and state

Plan file, `.scratch/phase-flow/<plan>.md` at the top of the checkout the run starts in (`git rev-parse --show-toplevel`; in a submodule, `git worktree list` names the git directory, not the checkout):

```markdown
# PUK-2749 puk-portal vulns
epic: PUK-2749
labels: CE+
## Phase 0
- ENG-9220
- ENG-9222
## Phase 1
- ENG-9225
- ENG-9221
```

`labels:` is optional: a comma-separated list of GitHub labels for every PR the run opens. Without it, PRs get no labels.

Stories are built in the order listed. Given an epic key, reuse `.scratch/phase-flow/<epic-lower>.md` if it exists; otherwise spawn the **Plan extractor** (briefs.md) and write its output there.

**Phase check.** Every story in a phase builds on the same base, which holds only the earlier phases. So every story still to build must have its "is blocked by" stories in an earlier phase, already merged, or Done; a story that already has an open or merged PR stays put. Before any phase starts, spawn the **Plan checker** (briefs.md) on the plan file. It moves each unbuilt story whose blocker sits in the same or a later phase into the phase after that blocker, cascading to the stories that wait on it, and leaves the order within each phase unchanged. Write its revised plan over the plan file, and record each move as a Decision. A blocker outside the plan that isn't Done or merged stays where it is, listed under Follow-ups; its story is built anyway. The checker's `stale:` lines (built stories whose blocker comes later, usually a link to delete) go under Follow-ups too.

State file, `.scratch/phase-flow/<plan>.state.md`, written only by the orchestrator:

- A table with one row per slice: `phase | key | slice | scope | after | branch | base | PR | status | round | note`. The status is one of `todo`, `building`, `built`, `reviewing`, `reviewed`, `ready`, `blocked`. Set the status before a step starts and update the row as soon as it ends. `round` is the pr-loop round in progress.
- `## Tmp branches`: each pushed `tmp-<plan>-phase-<n>` and its head sha.
- `## Decisions`: one line per decision taken in place of the user, with its story or phase.
- `## Follow-ups`: pre-existing bugs, accepted CI failures, SEPARATE_PR items and leftover findings.

**Existing work** (usually phase 0): before building a story, look for its PRs (`gh pr list --state all --search "<KEY> in:title"`) and a ticket-flow slice plan (`.scratch/ticket-flow/<KEY>.md`). An open PR is a built slice: record it as `built` and carry on with its review and CI. A merged PR is a slice already in the default branch: record it as `ready` and leave it out of tmp merges. Closed PRs don't count. A split story whose slices don't all have an open or merged PR still has slices to build. A story already Done in Jira with no open PR is skipped.

## 0. Preflight

- Ensure `.scratch/` is in `.git/info/exclude`. Refuse on a dirty tree. `git fetch origin`.
- Read the plan, run the phase check, and create the state file if missing. Seed it with one row per story, filled in from Existing work.
- With `plan`: print the phase table (phase, key, existing PRs and status, moves made by the phase check), the external blockers, the stale links, the labels the PRs will get (or `none`) and the state file's path, then stop. Nothing else changes: no label creation, no Jira, no branches.
- Create each of the plan's labels the repo lacks (`gh label create "<label>"`).
- Note the default branch, the repo's PR title convention (CLAUDE.md, recent merged PR titles), the PR template, the user's Jira accountId (`atlassianUserInfo`), and which events the CI workflows trigger on (`.github/workflows`).

## 1. Phase base

Phase 0 builds on the default branch. Phase n > 0 builds on `tmp-<plan>-phase-<n-1>`. If that branch exists on the remote, reuse it as it is. If it doesn't (a run started with `from`), build the tmp branches for phases 0..n-1 in order with step 3, after finding their stories' PRs as under Existing work. If a story in those phases has no open or merged PR, stop and report.

## 2. Each story of the phase, one at a time

Skip stories whose slices are all `ready` or `blocked`. Work through a story's slices in order, each one all the way to `ready` (or `blocked`) before the next starts, so a dependent slice branches from its reviewed predecessor.

1. **Start.** Assign the story to the user if unassigned. Move it to In Progress: `getTransitionsForJiraIssue`, then the transition whose target status is "In Progress". If the story is already past In Progress, or no such transition exists, record a Decision and go on.
2. **Build.** Spawn a fresh **Builder** (briefs.md) for the slice. Pass it the story key, the slice (number, scope, criteria; `whole story, decide the split` for an unsplit story), the base (the phase base, or the branch of the slice named in `after`), the branch (`<convention>/<key-lower>-<slug>`, `-<n>-<slug>` for slice n), and whether the branch or PR already exists. On a story's first Builder, write the slice plan it returns into the table. It returns the PR number, decisions, test results and anything left out, or `BLOCKED: <reason>`. Comment the PR link on the story (`PR: <url>`, or `PR <n>/<total>: <url>`), and copy the decisions into the state file.
3. **Review.** Check `git status --porcelain` is empty, then run the review loop on the PR as pr-loop does: read `../pr-loop/SKILL.md` and follow it with `afm-skills:pr-review <n> auto` and `afm-skills:resolve-pr-comments <scratch-path> auto`, 5 rounds at most, updating `round` each round. SEPARATE_PR items and leftover low findings go under Follow-ups. If the loop hits the cap with medium-or-higher findings open, keep the PR in draft, list them under Follow-ups and mark the slice `blocked`. Otherwise mark it `reviewed`.
4. **CI.** Wait for the checks in a background Bash (`gh pr checks <n> --watch`) and act on its completion notice. If no checks register within 5 minutes, the workflows don't run on this base: record that under Follow-ups, and run the repo's CI commands locally with the **Merge verifier** brief on the PR branch instead. On a failure, spawn the **CI fixer** (briefs.md). It either pushes a fix, or proves the failure pre-existing: the same check fails the same way on the base. A pre-existing failure is accepted: record it under Follow-ups and in the PR body's CI section. Wait again after each fix. Allow 3 fixer attempts per PR, then mark the slice `blocked` with the failing checks.
5. **Hand off.** When every check is green or accepted, `gh pr ready <n>`, status `ready`. When every slice of the story is `ready`, move the story to its review status: the transition whose target name contains "Review" and whose status category is In Progress (e.g. "Ready for Review (dev)", "PR For Review (Dev)"). If none matches, record a Decision.

## 3. Phase end: the tmp branch

When every slice of the phase is `ready` or `blocked`:

- If any slice is `blocked`, stop the run and report: the next phase builds on all of them.
- Otherwise build `tmp-<plan>-phase-<n>` in a separate worktree in the scratchpad, so the main checkout stays put. Start from the phase base and merge every open PR head of the phase in plan order (`git merge --no-ff origin/<head>`), every slice included; git skips history the slices share. On a conflict, spawn the **Merge resolver** (briefs.md) and record its per-file lines under Decisions.
- Verify the result with the **Merge verifier** (briefs.md). A failure caused by the merge goes back to the resolver. A failure that already exists on one of the merged PRs or on the phase base is accepted: record it under Follow-ups.
- Push it, and record it under Tmp branches. Once a tmp branch has PRs based on it, it is never rebuilt or force-pushed.

Then start the next phase at step 1. After the last phase, build its tmp branch too, so the user has one branch with the whole plan applied.

## 4. Report

At the end, or when the run stops: the state table, the Tmp branches, the Decisions, the Follow-ups, and what the user still has to do by hand (Jira links the MCP can't delete, staging checks, required-check settings, merging). Follow-ups stay in the state file; open no Jira issues for them.
