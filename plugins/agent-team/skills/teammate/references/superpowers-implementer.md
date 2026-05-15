# Superpowers development workflow — implementer teammate role

This is a reference document for the `agent-team:teammate` skill. Read it when your spawn brief signals you're an implementer in the full `superpowers` development workflow (typical signals: a worktree path under `.claude/worktrees/`, references to a spec doc and a plan doc).

Preconditions:

- `agent-team:teammate` is already active. That's the team primitive — identity, inbox sync, dispatch limits. This doc assumes you've read it and won't re-cover any of it.
- The `superpowers` plugin is on the host.
- The lead's spawn prompt gave you a worktree path, a spec, and a plan.

This doc maps the superpowers' skill triggers onto your work, and tells you when to dispatch subagents vs. work in your own context.

(Reviewer teammates have a separate doc: `references/code-review.md`.)

## The pattern: orchestrate, don't code

Your work for the rest of the session is **dispatching subagents**, not direct coding. Implementers, debuggers, reviewers — all run as subagents you spawn, brief, and orchestrate. You read their results, decide what to do next, and dispatch the next subagent.

Direct edits / direct file writes from you are limited to: small follow-up touches that don't need a full subagent round-trip (typo fixes, a single-line clarification), and integration glue between subagent outputs.

This is the `superpowers:subagent-driven-development` pattern. Internalize it before Step 4 work begins.

## Step 4 — startup

After the first actions from `agent-team:teammate` (ToolSearch, read team config, drain inbox), do these:

### 4.1 Unconditional triple-invoke

Invoke these three skills *before* reading project docs or doing task work:

1. `Skill('superpowers:using-superpowers')` — meta-skill that governs all the others.
2. `Skill('superpowers:subagent-driven-development')` — the orchestrate-don't-code pattern.
3. `Skill('superpowers:dispatching-parallel-agents')` — for any fan-out of subagents.

Invoke them all up front, without waiting for triggers. The reason: setting the orchestration discipline before the first task starts prevents you from sliding into direct-coding habits when work pressure arrives.

### 4.2 Enter your worktree

```
EnterWorktree(path=".claude/worktrees/<your-branch>")
```

The lead pre-created this in their Step 3.2. Your `name`, branch name, and worktree basename are identical by convention — that consistency lets filesystem-based detection (and later hook automation) identify your session.

### 4.3 Read the brief and the static docs

In order:

- The brief in your spawn prompt (already in context).
- The project overview (`overview.md` or whatever path the brief names).
- Your module's spec.
- Your module's plan.

Understanding these well saves you many subagent round-trips later — the subagents you dispatch won't have read them and will produce confused output if you can't summarize the relevant constraints in their prompts.

### 4.4 The work loop

Per-step work loop, where each "step" is one task in your share of the task list:

```
do the step's work (Steps 5–6 for implement+review; Step 7 for PR)
→ TaskUpdate(status="completed")
→ Read your inbox
→ branch per agent-team:teammate's inbox-sync rules
```

Initiate `SendMessage` to the lead only when: BLOCKED, all your tasks done, replying to an injected message, or sending `shutdown_response`.

## Steps 5–6 — per-task implementer + two-stage review (run via `superpowers:subagent-driven-development`)

For every implementation task, follow `superpowers:subagent-driven-development`. That skill prescribes the whole pattern — implementer subagent → spec reviewer subagent → code-quality reviewer subagent → fix loop — and you should read it through once and execute it as written.

Three team-specific additions on top of the skill:

1. **Implementer subagent prompts must name the discipline skills + their triggers.** Inside your dispatch prompt, do not write "use TDD" / "self-review when done". Subagents pseudo-comply with vague instructions (skipping the RED phase, fabricating verification). Instead, name skills and trigger conditions:

   ```text
   Skills to invoke (explicit):
   - On start: Skill('superpowers:test-driven-development').
   - On any bug / test failure: Skill('superpowers:systematic-debugging').
   - Before claiming DONE: Skill('superpowers:verification-before-completion').

   In your DONE report, list which skills you invoked and one line of what each contributed.
   ```

2. **Reviewer-subagent dispatch goes through `superpowers:requesting-code-review`.** Use that skill's template for both the spec-compliance reviewer and the code-quality reviewer.

3. **Treat every reviewer issue through `superpowers:receiving-code-review`.** First invocation lands here in Step 6; the skill remains in effect for the Step 9 fix loop with the reviewer teammate.

Re-dispatch the implementer subagent for fixes; re-dispatch the reviewer subagents for re-review. Loop until both reviewers return clean, then mark the task complete and start the next one.

## Step 7 — wrap-up and PR

Once Steps 5–6 are clean across all your tasks:

1. **`superpowers:finishing-a-development-branch`** — pick Option 2 (Push and Create PR).
2. **`commit-commands:commit-push-pr`** — handles commit + push + PR creation. Follow its own commit-granularity guidance.
3. **Report the PR URL to the lead** with `SendMessage`. Do the before-`SendMessage` inbox check from `agent-team:teammate` first.
4. **Do not `ExitWorktree(action="remove")`.** You entered by `path`, and the lead handles worktree removal in their Step 10 (via `EnterWorktree(name="<branch>")` + `ExitWorktree(action="remove")` in their own session). Leave the worktree on disk, idle, wait.

## Step 9 — fix loop with the reviewer teammate

After your PR is open, the lead spawns a separate reviewer teammate (their Step 8). When the reviewer messages "I posted review on PR #N, go pull them":

1. Pull the full comment thread: `gh api repos/{owner}/{repo}/pulls/{N}/comments` (not just the top-level summary).
2. `superpowers:receiving-code-review` is still in effect from Step 6 — apply it to each comment (verify before acting, push back when wrong).
3. Dispatch subagents the same way as Steps 5–6 for the fix work — `systematic-debugging` to investigate, implementer to apply the fix, then spec + code-quality reviewers to self-review before pushing.
4. `git commit` + `git push`. `SendMessage` the reviewer teammate: "Pushed fix for issues X, Y. Please re-review." Before-`SendMessage` inbox check first.
5. Reviewer's next PR comment: `**Changes requested**` → loop. `**APPROVED**` → `SendMessage` the lead with final status, then idle.

## Related docs and skills

- `agent-team:teammate` — required peer; team primitive. Invoke first.
- `skills/teammate/references/code-review.md` — sibling doc for reviewer teammates.
- `skills/lead/references/superpowers-workflow.md` — the lead-side workflow doc (orchestration Steps 1-3, 8, 10).
- `superpowers:using-superpowers` / `subagent-driven-development` / `dispatching-parallel-agents` — invoked up front in Step 4.1.
- `superpowers:test-driven-development` / `systematic-debugging` / `verification-before-completion` — named in implementer subagent prompts per Step 5/6.
- `superpowers:requesting-code-review` / `receiving-code-review` — invoked directly by you in Step 6 (and `receiving-code-review` re-applies in Step 9).
- `superpowers:finishing-a-development-branch` / `commit-commands:commit-push-pr` — invoked in Step 7.
