# afm-skills

Personal Claude Code skills and agents.

## PR workflow

- `/afm-skills:pr-review [pr] [post|auto]`: four reviewers run in parallel (two thermo-nuclear subagents, ponytail-review and an adversarial pass). It dedupes their findings, you approve each comment, and the result goes to `.scratch/pr-review/<pr>.md`. With `post`, it goes on the PR instead.
- `/afm-skills:resolve-pr-comments [pr|findings.md] [auto]`: a subagent judges each comment, then the plugin applies the fixes, pushes and replies.
- `/afm-skills:pr-loop [pr]`: runs review → resolve until no finding of medium severity or above is left (5 rounds max).

Needs: `gh` installed and signed in.

## Ticket workflow

- `/afm-skills:ticket-flow [auto] <epic...|story>`: picks the next user story in the epics. If the story is too big for one PR, it agrees a split into slices with you and records it in `.scratch/ticket-flow/<story>.md` (never as Jira subtasks). It assigns the story to you and moves it to In Progress, agrees the TDD seams for the next slice, builds that slice test-first in a subagent, opens a draft PR and links it in a Jira comment. Then it runs pr-loop and marks the PR ready. Once every slice has a ready PR, it moves the story to Ready for Review (dev). One run builds one slice. `auto` takes the recommended story without asking, but the split and the seams still need your OK. To work through whole epics, wrap it in a loop: `/loop /afm-skills:ticket-flow auto EPIC-1 EPIC-2`. The loop ends when no story has work left.

Needs: the Atlassian MCP connector, `gh`, and the mattpocock-skills plugin (for `tdd`).

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

## Credits

- [Andrea Cappadona](https://github.com/andreacappadona17) wrote the first version of `pr-review`.
- [Daniele Cammareri](https://github.com/dancamma) wrote the first version of `resolve-pr-comments`.
- The thermo-nuclear review agents are adapted from [cursor/plugins `thermos`](https://github.com/cursor/plugins/tree/main/thermos) (MIT, see [agents/LICENSE-thermos](agents/LICENSE-thermos)).
