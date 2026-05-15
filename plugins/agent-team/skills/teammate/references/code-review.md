# Superpowers development workflow — reviewer teammate role

This is a reference document for the `agent-team:teammate` skill. Read it when your spawn brief signals you're a reviewer in the full `superpowers` development workflow (typical signals: a PR URL, a name like `reviewer-pr-<N>`, no worktree path / spec / plan).

Preconditions:

- `agent-team:teammate` is already active. That's the team primitive — identity, inbox sync, dispatch limits. This doc assumes you've read it and won't re-cover any of it.
- The `code-review:code-review` skill is on the host (you'll invoke it once per review pass).
- The lead's spawn prompt gave you a PR URL and named you `reviewer-pr-<N>`.

(Implementer teammates have a separate doc: `references/superpowers-implementer.md`.)

## Your role

You review an implementer teammate's PR, post the result, and run a `SendMessage` fix loop with the implementer until the PR is `**APPROVED**`. You don't write code; you don't need a worktree (`code-review:code-review` operates on the PR diff via `gh`).

## Startup

After the first actions from `agent-team:teammate` (ToolSearch, read team config, drain inbox):

1. Note the PR URL and the implementer teammate's name from your spawn brief.
2. Skip `EnterWorktree` — you don't need a worktree.

## The review loop

Per round, invoke `Skill('code-review:code-review')` and let it run — that command already handles the 5-parallel-reviewer + Haiku confidence-score pipeline and produces a structured findings list. Your job is to wrap that output into the team's protocol:

1. **Run the review** — invoke the skill, get findings.
2. **Post the verdict to the PR** via `gh pr comment <PR#>`. The **first line of the body** must be one of:
   - `**APPROVED**` — no blocking issues; implementer can stop iterating.
   - `**Changes requested**` — issues remain.

   The implementer parses this first line to detect the verdict. Inline replies to existing review threads go through `gh api repos/{owner}/{repo}/pulls/{N}/comments/{id}/replies`, not new top-level comments.

   *Why `gh pr comment` instead of `gh pr review --approve` / `--request-changes`:* GitHub blocks PR authors from approving their own PRs, and in a solo-developer setup the reviewer and implementer share the host's `gh` identity (so the reviewer counts as the PR author). The `gh pr comment` convention works around that limit.

3. **Notify the implementer** with `SendMessage`: "Posted review on PR #N." Before-`SendMessage` inbox check from `agent-team:teammate` first.

4. **Idle** until the implementer pushes a fix and `SendMessage`s you to re-review. Loop back to step 1 against the new HEAD.

5. **After posting `**APPROVED**`**, `SendMessage` the lead with the final status, then idle and wait for shutdown. Before-`SendMessage` inbox check first.

## Receiving pushback from the implementer

The implementer may push back on individual review items per `superpowers:receiving-code-review`. Treat their pushback as input, not as an attack — they may have context you missed (existing conventions, deliberate tradeoffs, prior decisions). Apply the same evaluation pattern: read their reasoning, verify against the codebase, decide whether to drop the issue from your next review pass or restate it with new evidence.

## Related docs and skills

- `agent-team:teammate` — required peer; team primitive. Invoke first.
- `skills/teammate/references/superpowers-implementer.md` — sibling doc for implementer teammates.
- `skills/lead/references/superpowers-workflow.md` — the lead-side workflow doc.
- `code-review:code-review` — the actual review skill you invoke per loop iteration.
- `superpowers:receiving-code-review` — describes how the implementer evaluates your feedback (useful context when they push back).
