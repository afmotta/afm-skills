---
name: ticket-flow
description: Take one ticket of a Jira epic from pick to review-ready. Pick the next ticket, move it to In Progress, build it test-first with the tdd skill, open the PR, run /pr-loop, then move it to Ready for Review (dev). Subagents do the reading and coding. Use when the user invokes /ticket-flow with an epic or ticket key.
disable-model-invocation: true
argument-hint: "<epic-key|ticket-key>"
---

# Ticket flow

Target: `$ARGUMENTS`. Jira goes through the Atlassian MCP tools (get the cloudId once with `getAccessibleAtlassianResources`). GitHub goes through `gh`.

The main context only orchestrates. It keeps keys, short summaries and the PR number. Ticket bodies, code and diffs stay inside subagents.

The user is asked twice: to pick the ticket and to confirm the seams to test. Everything else runs unattended.

## 0. Preflight
- Refuse on a dirty tree. Note the default branch (`gh repo view --json defaultBranchRef -q .defaultBranchRef.name`).
- Fetch the target. If it's an epic, go to 1. If it's a ticket that isn't Done, skip to 2.

## 1. Pick (subagent)
Spawn a `general-purpose` subagent with this brief: list the epic's open children (JQL `parent = <epic> AND statusCategory != Done`) with status, priority, rank and assignee, plus their issue links (blocks / is blocked by). Check whether each blocker is Done. Look at the repo only if it settles a tie. Return at most 3 candidates, best first: key, title, a one-line reason and any open blocker. Leave ticket bodies out of the reply.

Ask the user with AskUserQuestion, with the recommended candidate first.

## 2. Start
- Assign the ticket to the user: get their accountId with `atlassianUserInfo`, then call `editJiraIssue` with `assignee: {accountId}`. Skip this if they're already the assignee.
- Move the ticket to **In Progress**: call `getTransitionsForJiraIssue` and take the transition whose target status matches "In Progress" (case-insensitive), then call `transitionJiraIssue`. If none matches, list the available transitions and ask.
- `git switch <default> && git pull --ff-only && git switch -c <KEY>-<short-slug>`. If recent remote branches follow a different naming convention, use theirs.

## 3. Seams (subagent)
Spawn one `general-purpose` subagent (the "builder") with the ticket key and the branch. The builder reads the ticket (description, acceptance criteria, comments) and the code it touches, and loads the `mattpocock-skills:tdd` skill. It writes no code yet. It returns: a plan of up to 3 lines, the seams it proposes (interface plus the behaviors to test at each one), and any open questions.

Show the seams to the user and get them confirmed or edited. TDD forbids tests at unconfirmed seams.

## 4. Build (same builder, via SendMessage)
Send the builder the confirmed seams and these instructions:
- Run the tdd loop one vertical slice at a time and commit after each green, following the repo's commit convention.
- At the end, run the repo's full tests, lint and typecheck.
- Push, then `gh pr create` with title `<KEY>: <summary>` and a body covering what changed and why, the Jira link and how it was tested.
- Return the PR number and URL, the commits, the test command and its result, and anything left out.
- If it gets blocked, return `BLOCKED` with its questions and stop.

On `BLOCKED`, ask the user and send the answer to the builder with SendMessage.

Once the PR is open, comment on the ticket with `addCommentToJiraIssue`: `PR: <url>`.

## 5. PR loop (main context)
pr-loop has `disable-model-invocation`, so read `../pr-loop/SKILL.md` (relative to this skill's base directory) and follow it with the PR number. It runs here, not in a subagent, because pr-review spawns its own subagents and subagents can't nest.

## 6. Hand off
- If the loop stopped clean or with no fixes, and pr-loop reported no failed checks, move the ticket to **Ready for Review (dev)** the same way as in step 2.
- If it hit the round cap or checks are failing, leave the status alone and ask the user.
- Report: the ticket, the PR link, the Jira status, pr-loop's stop reason, and its SEPARATE_PR items as ticket candidates.
