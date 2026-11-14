# phase-flow subagent briefs

The orchestrator fills in the `<...>` fields. Every brief also gets the repo path, the default branch, the PR title convention, and its base branch: the phase base, or the earlier slice's branch for a stacked slice.

All subagents are `general-purpose` and **unattended**: nobody will answer a question. When something is open, settle it from the story, the epic, the code and the repo's CLAUDE.md, write the decision down, and go on. Return `BLOCKED: <reason>` only when those sources can't settle it, or when progress needs credentials, an external system or a human. Return text only; the orchestrator writes the state file.

Shared rules for every brief that runs code:

- Work in your own git worktree in the scratchpad, never in the main checkout. Leave the worktree clean and remove it (`git worktree remove`) before returning.
- Use the Node version the repo pins (`.node-version`, `engines`, `mise.toml`), e.g. through nvm; the shell's default Node may be older.
- Read the repo's CLAUDE.md first; its conventions win over this brief.
- Write nothing to shared environments. If `.env` points at a shared dev database or cache, point local runs at a local stack, or run only what needs neither.
- Make checks pass by fixing code. Tests, coverage thresholds, lint config and workflows change only when the story is about them; hooks always run.
- Commit with the repo's commit convention, staging files by name.

## Plan extractor

Read epic `<epic-key>` (getJiraIssue, markdown, with description and comments) and its child stories. Find the phase plan in the description: a section of phases or waves listing story keys in order. Return only the plan file content in this shape, keys in the order the epic lists them:

```
# <epic-key> <epic summary>
epic: <epic-key>
## Phase 0
- <KEY>
```

If the epic lists no phases, derive them from the stories' "is blocked by" links (phase 0 = no open blockers, phase n = blockers only in earlier phases) and add a first line `derived: blocks links`.

## Builder

You build `<slice>` of Jira story `<KEY>` in `<repo>`, unattended, and open its draft PR. Base: `<base>`. Branch: `<branch>` (`<exists: new | branch exists | PR #n exists>`; if it exists, continue from its commits).

1. **Read** the story (description, acceptance criteria, "Done when", comments), its epic's description (conventions such as audit counts in PR bodies), the code it touches, and what `<base>` already contains.
2. **Split**, when the slice is `whole story, decide the split`. Decide whether the story fits one PR that a reviewer reads in one sitting with one concern. If not, split it into PR slices that are each mergeable on their own, cover named criteria and together cover all of them, with as few `after` dependencies as possible. Build only slice 1, and return the slice plan.
3. **Seams.** Load `mattpocock-skills:tdd` and follow it, with one change: you choose the seams yourself, and the list you write in the PR body's Decisions section stands in for the user's confirmation. For an upgrade, the seams are mostly the existing suites that must stay green and unchanged, plus a test for any behaviour the upgrade changes.
4. **Build** test-first, one tracer bullet at a time, committing after each green. Measure before and after whatever the story or epic asks to measure (audit counts, build time, bundle size).
5. **Check** with the repo's full unit suites and CI's flags, then lint, typecheck and build. Compare any failure against `<base>`: a failure the base also has is pre-existing. Report it rather than fixing it here.
6. **PR.** Push, then `gh pr create --draft --base <base> --label "CE+" --assignee @me`.
   - **Title:** follow the repo's convention, with `<KEY> (<n>/<total>)` in it for a split story.
   - **Body:** follow the repo's PR template. Cover what changed and why, the Jira link, the slice (n/total) and what the other slices cover, how it was tested, the measurements, and that it is stacked on `<base>`.
   - **Ending:** close the body with **Decisions taken without review**: the split, the seams, and every judgment call, one line each.

Return in under 200 words: the PR number and URL, the commits, the test commands and results, the slice plan (title, scope, criteria and `after` per slice) if you split, the decisions, and anything left out. Or `BLOCKED: <reason>`.

## CI fixer

PR `<n>` (head `<head>`, base `<base>`) has failing checks: `<check names>`. Make CI green, or prove that the failure isn't this PR's.

1. Read the failing job logs (`gh run view <run> --log-failed`) and find the first real error.
2. Reproduce it on the PR head with CI's environment: the same Node version, `CI=true`, `TZ=UTC`, the same flags and worker count, and the same resource limits (for example `NODE_OPTIONS=--max-old-space-size=<CI's default>` when the log shows a heap limit).
3. Run the same thing on `<base>`. The failure is **pre-existing** in either of these cases:
   - the base fails the same way;
   - it depends on time, order or flakiness: a run with a shifted clock, a rerun or a different seed shows it.

   For a pre-existing failure, rerun the failed jobs (`gh run rerun <run> --failed`) and return how to reproduce it.

   Otherwise it is **this PR's**. Fix it with the smallest change, add a test if the fix is logic, push, and add a line to the PR body's CI section.

Return in under 150 words, for each failing check: its cause, then `fixed <sha>` or `pre-existing` with the evidence and a reproduction.

## Merge resolver

Building `<tmp-branch>`: merging `<head>` (PR `<n>`, `<KEY>`) conflicts in `<files>`. The PRs already merged into this branch are `<list>`.

Resolve each file so both PRs' intent survives:

- **Same config or setting:** where the PRs really compete (two versions of the same config or setting), keep the side of the story that owns that concern (for example, the coverage-baseline story owns the coverage config). Carry over anything the other side adds that doesn't conflict.
- **Same file added:** where both sides add the same file, combine their contents.

Then commit the merge.

Return one line per file: which side won, what was carried over, and why.

## Merge verifier

On `<branch>` in a fresh worktree:

1. Install dependencies (`npm ci` or the repo's equivalent).
2. Run the repo's full unit suites with CI's flags, then the build. Capture each exit code separately; don't let a pipe hide it.
3. For each failure, run the same thing on `<base>` and on the PR branch it came from.

Return per suite: pass or fail and the test counts. For each failure, give the file, the first error, and where else it fails (base, PR branch, or only here).
