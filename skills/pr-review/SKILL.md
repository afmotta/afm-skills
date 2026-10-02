---
name: pr-review
description: Three-phase PR review. Fan out to the two thermo-nuclear reviewers, /ponytail-review and an adversarial subagent, dedupe findings by severity, then draft each PR comment one at a time for approval, and finally write them to a scratch file — or post them as a PR review when the args say `post`. Use when the user says "review this PR", "pr-review", or invokes /pr-review with a PR number, URL or branch.
argument-hint: "[pr-number|url|branch] [post|auto]"
---

# PR Review

Target: `$ARGUMENTS` (PR number, URL or branch; default: the PR for the current branch via `gh pr view`).

Nothing is posted to GitHub unless the args contain `post`. Without it the review lands in a local scratch file — the mode for your own PRs.

**With `auto` in the args.** No user gates: skip the Phase 1 wait and Phase 2. Every new finding becomes an approved comment as-is; go straight to Phase 3's scratch-file delivery (`post` is ignored), under a `# Round <date-time>` heading. End by printing the count per severity. If nothing is new, write nothing and say so.

## Phase 1: findings

Gather the context once: PR title and body, `gh pr diff <n>`, the list of changed files, and the scratch file `.scratch/pr-review/<pr-number-or-branch>.md` if it exists. Every reviewer gets all of it under `### PR`, `### Git / diff output`, `### Changed files` and `### Prior findings`.

Launch the four reviewers in one message, in parallel:
- `subagent_type: pr-tools:thermo-nuclear-review-subagent` — bugs, breakage, security, devex, scoped to the diff.
- `subagent_type: pr-tools:thermo-nuclear-code-quality-review-subagent` — structure and maintainability.
- `subagent_type: general-purpose` — run the `ponytail:ponytail-review` skill on the diff (if the ponytail plugin is not installed, review the diff for over-engineering instead: reinvented stdlib, needless dependencies, speculative abstractions).
- `subagent_type: general-purpose` — adversarial review of the changed files as a whole, not just the diff: how the new code interacts with what's around it, callers, edge cases, failure modes.

Each returns findings as `severity (critical / high / medium / low) — file:line — issue — evidence`. A finding already in Prior findings is not new, whatever its verdict — drop it, unless the code changed since and the issue is genuinely different.

Deduplicate across reviewers (a finding raised by several is weighted up), resolve disagreements with your own reading of the code, and list them by severity.

Stop here and wait for the user to validate the list.

## Phase 2: draft comments, one at a time

From the list of findings we validated, proceed iteratively.
Draft a comment you would attach in a PR review, linked to a line or a set of lines of code.
Each comment must be self-conclusive, without referencing other comments. If you need to reference other parts of code, prefer permalinks or a brief description a human can understand without effort. Avoid being overly verbose.
If in the discussion a new finding arises, enqueue it to the list of comments to review.
Give a preview of the comment and wait: the user either asks for changes or approves it.
When approved, enqueue it to the list of approved comments, and proceed with the preview of the next comment.
Loop until no comments/findings are left.

## Phase 3: deliver

Reread all approved comments. If there are no issues in them:

**Default — scratch file.** Write them to `.scratch/pr-review/<pr-number-or-branch>.md` in the repo (create the directory if needed), appending a new dated section if the file already exists. Format each comment as:

```markdown
## [ ] <severity> — <file>:<line-range>

<permalink to the lines>

<comment body>
```

Then tell the user the path, and that `/resolve-pr-comments <path>` acts on it. Do not run `gh pr review`, `gh api ... /reviews` or any other command that posts to GitHub.

**With `post` in the args.** Append all comments to the PR review (`gh api` review with inline comments, or `gh pr review --comment`), without mentioning AI tools. Confirm with the user before posting.
