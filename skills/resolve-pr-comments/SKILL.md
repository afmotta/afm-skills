---
name: resolve-pr-comments
description: Work through the new review comments on a PR, or the findings in a pr-review scratch file. Each comment is judged by an independent subagent (fix / no fix needed / not worth fixing / separate PR / disagree), then the fixes are applied and pushed and every reviewer gets a short reply. Use when the user wants to address, resolve or answer PR review comments, or invokes /resolve-pr-comments. Also use to act on a .scratch/pr-review/*.md findings file.
user-invocable: true
argument-hint: "[pr-number|url|findings.md] [auto]"
---

<what-to-do>

Resolve the review comments on a PR. The subagents judge, you act and reply.

## 1. Load

- PR: the number or URL in the args; otherwise gh pr view --json number,title,body,headRefName,baseRefName. Repo: gh repo view --json nameWithOwner. Me: gh api user -q .login.
- Check out the PR head branch (`gh pr checkout <n>`) if not on it. Refuse to continue with a dirty working tree.
- Read the PR title/body and gh pr diff <n> once: the subagents get this as context.

## 2. Collect the new comments

**Scratch-file source.** When the args contain a path to a markdown file (a `.scratch/pr-review/*.md` written by `pr-review`), read the findings from it instead of from GitHub: each `## <severity> — <file>:<lines>` section is one comment, its body is the comment body, and there is no reviewer. A finding is new when its checkbox is unticked. Skip the rest of this step. The user already approved these findings once, so step 3 is a second opinion, not a gate: run it, but default to `auto` unless the args say otherwise.

A comment is new when the PR author has not answered it yet.

Inline threads (the main source):


gh api graphql -F owner=<o> -F repo=<r> -F pr=<n> -f query='
query($owner:String!,$repo:String!,$pr:Int!){ repository(owner:$owner,name:$repo){ pullRequest(number:$pr){
  reviewThreads(first:100){ nodes{ id isResolved isOutdated path line
    comments(first:50){ nodes{ databaseId url author{login} body diffHunk createdAt } } } } } } }'


Keep threads where isResolved is false and the last comment's author is not me.

Top-level: gh api repos/<o>/<r>/pulls/<n>/reviews (non-empty body`) and `gh api repos/<o>/<r>/issues/<n>/comments. Keep the ones not by me and created after my last comment on the PR.

Nothing new: say so and stop.

## 3. Judge, one subagent per comment

Launch all subagents in one message: Agent, subagent_type: general-purpose. Each gets: PR title and body, the file path and line, the diffHunk, the full thread (earlier comments included), and the repository path. It reads the code around the comment and the relevant ADRs/`CLAUDE.md` before ruling. It never edits files.

It must return exactly this:

verdict: FIX | NO_FIX | NOT_WORTH | SEPARATE_PR | DISAGREE
reason: <one or two sentences, English>
fix_spec: <only for FIX: what to change, where, in 1-3 lines>
reply: <the reply to the reviewer, max 3 sentences>


Verdicts:
- FIX: the comment is right and the change belongs in this PR.
- NO_FIX: the comment is right but asks for nothing to change (a question, a note, something the code already handles). The reply answers it.
- NOT_WORTH: the comment is right but the cost is above the benefit here. The reply says why.
- SEPARATE_PR: the comment is right but out of scope for this PR.
- DISAGREE: the comment is not right. The reply explains, with a file/line or ADR reference when possible.

Tell the subagent: the default is to take the reviewer seriously; DISAGREE needs a concrete reason read off the code, not a preference. NOT_WORTH is not a way to avoid work.

## 4. Confirm

Show one table: comment (path:line, author, first line), verdict, reply. Ask once before you touch code or post anything. Skip this step only when the args contain auto.

## 5. Act

- FIX: apply the fix_spec, in the same style as the surrounding code. Run the checks that touch the file (the workspace test and `lint`). One commit per comment, subject in the repo's conventional style; when a fix would be several lines of reasoning, delegate it to a subagent with the spec. Push once at the end.
- SEPARATE_PR: no code. Collect these for the final report so the user can open the ticket. Agents never write Linear.
- Others: no code.

## 6. Reply

Scratch-file source: no GitHub calls at all. Tick each finding's checkbox and write the verdict and reply underneath it in the same file, `**<VERDICT>** — <reply>` (with the sha for FIX). Then skip to step 7.

Inline thread: gh api repos/<o>/<r>/pulls/<n>/comments/<databaseId>/replies -f body='...' on the last comment of the thread. Top-level: gh pr comment <n> --body '...', quoting the first line of the original in a > block.

For FIX, append the short commit sha to the reply. Do not resolve threads: the reviewer does that.

Reply style, short, plain, no thanks, no emoji, no preamble:
- FIX: Done in <sha>: <what changed in one sentence>.
- NO_FIX: <direct answer to the question or doubt>.
- NOT_WORTH: True, but <why it's not worth it here>. I leave it as is.
- SEPARATE_PR: Right, but out of scope for this PR. I will address it in a separate PR.
- DISAGREE: I am not convinced: <concrete reason, with reference to file/line or ADR>.

## 7. Report

To the user, only: the verdict table with the sha per FIX, the SEPARATE_PR items as ticket candidates, and anything a check failed on.

</what-to-do>