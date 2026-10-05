# afm-skills

Personal Claude Code skills and agents.

## PR workflow

- `/afm-skills:pr-review [pr] [post|auto]`: four reviewers run in parallel (two thermo-nuclear subagents, ponytail-review and an adversarial pass). It dedupes their findings, you approve each comment, and the result goes to `.scratch/pr-review/<pr>.md`. With `post`, it goes on the PR instead.
- `/afm-skills:resolve-pr-comments [pr|findings.md] [auto]`: a subagent judges each comment, then the plugin applies the fixes, pushes and replies.
- `/afm-skills:pr-loop [pr]`: runs review → resolve until no finding of medium severity or above is left (5 rounds max).

Needs: `gh` installed and signed in.

## Install

```
/plugin marketplace add afmotta/afm-skills
/plugin install afm-skills@afm-skills
```

Recommended: the ponytail plugin. pr-review uses its `ponytail-review` skill and falls back to a plain over-engineering pass without it.

```
/plugin marketplace add DietrichGebert/ponytail
/plugin install ponytail@ponytail
```
