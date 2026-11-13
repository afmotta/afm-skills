---
name: ticket-flow
description: Take one slice of a Jira user story from pick to review-ready. Pick the next story of one or more epics, split it into PR-sized slices if it's too big (tracked in a local scratch file, never as subtasks), move it to In Progress, build the next slice test-first with the tdd skill, open its PR, run /pr-loop, and move the story to Ready for Review (dev) once every slice has a PR. Subagents do the reading and coding. Use only when the user invokes /ticket-flow with epic or story keys, or a /loop the user started runs it. Never start it on your own: it moves Jira statuses and opens PRs.
argument-hint: "[auto] <epic-key...|story-key>"
---

# Ticket flow

Target: `$ARGUMENTS`. If the first word is `auto`, run in auto mode and drop it from the target. The rest is one story key or one or more epic keys. Jira goes through the Atlassian MCP tools (get the cloudId once with `getAccessibleAtlassianResources`). GitHub goes through `gh`.

Work stays at the story level: never create subtasks or other Jira issues. A story too big for one PR is split into slices. The split lives in a local scratch file, and each slice gets its own branch, PR and pr-loop. One run of this skill builds one slice.

The main context only orchestrates. It keeps keys, short summaries and the PR number. Story bodies, code and diffs stay inside subagents.

The user is asked up to three times: to pick the story (skipped in auto mode), to confirm the split (once per story) and to confirm the seams to test. Everything else runs unattended.

Run under `/loop` (e.g. `/loop /afm-skills:ticket-flow auto EPIC-1 EPIC-2`), each run takes one slice. When step 1 finds nothing left to do, end the loop: `ScheduleWakeup` with `stop: true`, or `CronDelete` for an interval loop.

## Slice state
- Plan file: `.scratch/ticket-flow/<KEY>.md` in the main worktree (the first path in `git worktree list`), so every worktree of the repo sees the same plan. It's written only when a story has more than one slice: one line per slice, `<n>. <title>: <scope> [criteria] (after <m>)`, with the `(after <m>)` part only when slice n needs slice m merged first.
- A story with no plan file is one slice. If it's In Progress and has PRs titled `<KEY> (<n>/<total>)` but no plan file, the plan was lost: ask the user.
- A slice is taken once its PR exists. List them with `gh pr list --state all --search "<KEY> in:title" --json number,title,state,isDraft,url`: the title is `<KEY> (<n>/<total>): …`, or `<KEY>: …` for a one-slice story.
- A slice's `after` dependency is met when that slice's PR is merged, since every slice branches from the default branch.

## 0. Preflight
- Ensure `.scratch/` is in `.git/info/exclude` (append if missing), then refuse on a dirty tree. Note the default branch (`gh repo view --json defaultBranchRef -q .defaultBranchRef.name`).
- Fetch the target. If it's epics, go to 1. If it's a story that isn't Done, skip to 2.

## 1. Pick (subagent)
Spawn a `general-purpose` subagent with this brief: list the epics' stories that still have work (JQL `parent in (<epics>) AND (statusCategory = "To Do" OR (status = "In Progress" AND assignee = currentUser()))`) with status, priority, rank and assignee, plus their issue links (blocks / is blocked by). Check whether each blocker is Done. For In Progress stories, read the slice state (see above), drop the stories whose every slice has a PR, and name the next slice without a PR and whether its `after` dependency is merged. Look at the repo only if it settles a tie. Prefer finishing an In Progress story over starting a new one. Return at most 3 candidates, best first: key, title, next slice (or "not split yet"), a one-line reason and any open blocker or unmerged dependency. Leave story bodies out of the reply.

Stories whose slices all have a PR drop out, so a loop never builds a slice twice.

- No candidates: report "no stories with work left in <epics>", end the loop if there is one, and stop.
- Auto mode: take the best candidate with no open blocker or unmerged dependency and say which one in one line. If every candidate is blocked, report them, end the loop if there is one, and stop.
- Otherwise ask the user with AskUserQuestion, with the recommended candidate first.

## 2. Split (subagent, once per story)
If the story is In Progress, its slices are already set: read the slice state (unless step 1 already named the next slice), take the next slice without a PR, and go to 3. If every slice has a PR, report that and stop. Otherwise spawn a `general-purpose` subagent with the story key and this brief: read the story (description, acceptance criteria, comments) and the code it touches, without writing code. Decide whether it fits one PR that a reviewer can read in one sitting and that has one concern. If it doesn't, split it into vertical slices. Each slice must be mergeable on its own (behind a flag if needed) and cover named acceptance criteria. Together the slices must cover every criterion. Order the slices and keep `after` dependencies to a minimum, because a dependent slice waits for its predecessor to merge. Never create Jira issues or subtasks. Return either "one slice" with a one-line reason, or the slices: title, scope line, the criteria each covers, and `after`.

Show the split to the user and get it confirmed or edited, in auto mode too. For more than one slice, write the plan file.

## 3. Start
- On the story's first slice: assign the story to the user (get their accountId with `atlassianUserInfo`, then call `editJiraIssue` with `assignee: {accountId}`; skip this if they're already the assignee). Then move it to **In Progress**: call `getTransitionsForJiraIssue` and take the transition whose target status matches "In Progress" (case-insensitive), then call `transitionJiraIssue`. If none matches, list the available transitions and ask.
- If a branch for this slice already exists locally or on the remote (an interrupted run), switch to it and tell the builder to continue from its commits. Otherwise run `git switch <default> && git pull --ff-only && git switch -c <KEY>-<short-slug>`, with `<KEY>-<n>-<short-slug>` for slice n of a split story. If recent remote branches follow a different naming convention, use theirs.

## 4. Seams (subagent)
Spawn one `general-purpose` subagent (the "builder") with the story key, the slice (its number, title, scope and criteria; the other slices are out of scope) and the branch. The builder reads the story (description, acceptance criteria, comments) and the code it touches, and loads the `mattpocock-skills:tdd` skill. It writes no code yet. It returns: a plan of up to 3 lines, the seams it proposes (interface plus the behaviors to test at each one), and any open questions.

Show the seams to the user and get them confirmed or edited. TDD forbids tests at unconfirmed seams.

## 5. Build (same builder, via SendMessage)
Send the builder the confirmed seams and these instructions:
- Build only this slice. Run the tdd loop one vertical slice at a time and commit after each green, following the repo's commit convention.
- At the end, run the repo's full tests, lint and typecheck.
- Push, then `gh pr create --draft` with title `<KEY>: <summary>` (`<KEY> (<n>/<total>): <summary>` for a split story) and a body covering what changed and why, the Jira link, which slice this is and what the other slices leave for later, and how it was tested.
- Return the PR number and URL, the commits, the test command and its result, and anything left out.
- If it gets blocked, return `BLOCKED` with its questions and stop.

On `BLOCKED`, ask the user and send the answer to the builder with SendMessage.

Once the PR is open, comment on the story with `addCommentToJiraIssue`: `PR <n>/<total>: <url>`, or `PR: <url>` for a one-slice story.

## 6. PR loop (main context)
pr-loop has `disable-model-invocation`, so read `../pr-loop/SKILL.md` (relative to this skill's base directory) and follow it with the PR number. It runs here, not in a subagent, because pr-review spawns its own subagents and subagents can't nest.

## 7. Hand off
- If the loop stopped clean or with no fixes, and pr-loop reported no failed checks, mark the PR ready (`gh pr ready <n>`). If every slice of the story now has a PR and none is a draft, move the story to **Ready for Review (dev)** the same way as in step 3. Otherwise leave it In Progress.
- If it hit the round cap or checks are failing, leave the PR in draft and the status alone, and ask the user.
- Report: the story, the slice (n/total), the PR link, the Jira status, pr-loop's stop reason, and its SEPARATE_PR items as candidates for new stories or slices. Don't create them.
