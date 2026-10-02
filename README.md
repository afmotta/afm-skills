# pr-tools

Claude Code plugin with three skills:

- `/pr-tools:pr-review [pr] [post|auto]`: four reviewers run in parallel (two thermo-nuclear subagents, ponytail-review and an adversarial pass). It dedupes their findings, you approve each comment, and the result goes to `.scratch/pr-review/<pr>.md`. With `post`, it goes on the PR instead.
- `/pr-tools:resolve-pr-comments [pr|findings.md] [auto]`: a subagent judges each comment, then the plugin applies the fixes, pushes and replies.
- `/pr-tools:pr-loop [pr]`: runs review → resolve until no finding of medium severity or above is left (5 rounds max).

Needs: `gh` installed and signed in.

## Install

```
/plugin marketplace add <github-user>/pr-tools
/plugin install pr-tools@pr-tools
```

Recommended: the ponytail plugin. pr-review uses its `ponytail-review` skill and falls back to a plain over-engineering pass without it.

```
/plugin marketplace add DietrichGebert/ponytail
/plugin install ponytail@ponytail
```
