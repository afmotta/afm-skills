---
name: pr-loop
description: Loop /pr-review and /resolve-pr-comments on a PR until a review round finds nothing new of medium severity or above (max 5 rounds). Fully autonomous; findings stay in a local scratch file, fixes are committed and pushed. Use when the user invokes /pr-loop or asks to review-and-fix a PR until clean.
disable-model-invocation: true
argument-hint: "[pr-number|url|branch]"
---

# PR loop

Target: `$ARGUMENTS` (default: the PR for the current branch, `gh pr view`).

## Setup
- Resolve the PR number; `gh pr checkout <n>` if not on its head branch. Refuse on a dirty tree.
- Ensure `.scratch/` is in `.git/info/exclude` (append if missing).
- Scratch file: `.scratch/pr-review/<pr-number>.md`.

## Each round (max 5)
1. **Review.** Invoke the `afm-skills:pr-review` skill with `<n> auto`.
2. **Stop check.** No new finding of medium or above → Report. This round's low findings stay unticked in the scratch file.
3. **Resolve.** Invoke the `afm-skills:resolve-pr-comments` skill with `<scratch-path> auto`.
4. No finding got a FIX verdict → the code didn't change, another review would see the same thing → Report. Otherwise next round.

## Report
One table across all rounds: round, severity, file:line, verdict, sha. Then: why it stopped (clean / no fixes / cap hit), SEPARATE_PR items as ticket candidates, unticked low findings (`/resolve-pr-comments <scratch-path>` handles them), failed checks.
