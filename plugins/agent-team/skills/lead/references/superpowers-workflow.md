# Superpowers development workflow — lead role

This is a reference document for the `agent-team:lead` skill. Read it when running the full `superpowers` development workflow (brainstorming → writing-plans → parallel implementer teammates → reviewer teammates → PR fix loop → user-approved merge).

Preconditions:

- `agent-team:lead` is already active. That's the team primitive — TeamCreate, spawn rules, hub-and-spoke, shutdown sequence, merge order. This doc assumes you've read it and won't re-cover any of it.
- The `superpowers` plugin is installed on the host.
- The user wants the full discipline: brainstormed spec → written plan → TDD implementation → two-stage code review → PR ↔ reviewer fix loop → user-approved merge.

This doc maps the `superpowers:*` skill triggers onto the team's lifecycle.

## Skills you'll invoke directly

These fire in Steps 1-3 below — invoke them by name when their step arrives and follow them as written. This doc only highlights team-specific *additions* to each skill, not the skill's own process.

- `superpowers:brainstorming`
- `superpowers:writing-plans`
- `superpowers:using-git-worktrees`

The other `superpowers:*` and `code-review:code-review` skills run *inside* teammates (per the teammate-side reference docs), not in your context.

## Step 1 — brainstorming

Invoke `superpowers:brainstorming`. No team-specific deviation — just follow the skill.

## Step 2 — writing-plans

Invoke `superpowers:writing-plans`. Two team-specific additions on top of the skill's standard plan structure:

- **Explicit parallel split.** The plan names which tasks are independent enough to run as separate teammates. This is the input to Step 3's spawn count.
- **Branch names per teammate.** Each parallel slot gets a branch name. The branch name becomes the teammate's `name` *and* the worktree directory basename — keep all three identical, so filesystem-based detection (and later hook automation) can key off the `cwd` basename.

## Step 3 — worktrees + TeamCreate + spawn

### 3.1 Invoke `superpowers:using-git-worktrees`

It governs the `EnterWorktree` / `ExitWorktree` flow you're about to use.

### 3.2 Create one worktree per teammate (serial loop)

Only one worktree can be active per session, so you create them one at a time and "keep" each on disk before moving to the next:

```
for each teammate (per the plan's split):
    EnterWorktree(name="<branch-name>")    # creates and enters
    ExitWorktree(action="keep")            # leaves worktree on disk; you return to main
```

After the loop: all worktrees exist at `.claude/worktrees/<branch-name>/`, and you're back on the main worktree / main branch.

### 3.3 `TeamCreate` + `TaskCreate`

Standard `agent-team:lead` mechanics — create the team, populate the shared task list.

### 3.4 Spawn implementing teammates in parallel

Issue all `Agent` calls in a single message (one message → parallel; multiple messages → serial). Each spawn prompt follows the standard `agent-team:lead` structure, plus three superpowers-specific elements in the task block:

1. **Worktree path** — tell the teammate exactly which path to `EnterWorktree(path=".claude/worktrees/<branch>")`. They join the worktree you pre-created.
2. **Docs to read** — `overview.md`, the module's spec, the module's plan.
3. **Startup instruction** — after `Skill('agent-team:teammate')` (the standard first action), tell the teammate to Read its own skill's `references/superpowers-implementer.md`. That doc covers Steps 4-7 + their side of Step 9.

A typical implementer spawn prompt:

```text
You are teammate "{name}" in team "{team_name}", role: {role}.

First actions, in order:
1. Skill('agent-team:teammate') — team coordination protocol.
2. Read your skill's references/superpowers-implementer.md — the per-task TDD +
   two-stage review + PR pattern you'll be running for the rest of the session.

Project context:
- Worktree:        .claude/worktrees/{branch}   (lead pre-created; EnterWorktree(path=...))
- Overview:        {path}
- Your spec:       {path}
- Your plan:       {path}
- Other teammates: {names + roles + worktrees}

Your task:
{specific module, success criteria, dependencies on other teammates}

Coordination resources:
- Team config: ~/.claude/teams/{team_name}/config.json
- Inbox file:  ~/.claude/teams/{team_name}/inboxes/{name}.json
```

## Monitoring while teammates run Steps 4–7

Steps 4–7 happen *inside* the implementing teammates (TDD per task → two-stage review → PR). Your job during this phase:

- Respond to `SendMessage`s — progress reports, BLOCKED requests, eventually PR URLs.
- `SendMessage` direction changes if the human user updates the plan. Teammates' inbox-sync protocol (from `agent-team:teammate`) picks the change up at their next step boundary.
- Resist peeking at worktrees / log files. `TaskList`, `TaskGet`, and incoming `SendMessage` are the canonical visibility surface.

## Step 8 — spawn a reviewer teammate per PR

When an implementer teammate `SendMessage`s you with a PR URL, spawn a **separate** reviewer teammate against that PR. Same spawn-prompt structure as in Step 3.4, with:

- `name="reviewer-pr-<N>"`.
- The brief tells the reviewer: the PR URL, first action `Skill('agent-team:teammate')`, then Read its skill's `references/code-review.md` (which guides the review loop).
- No worktree — `code-review:code-review` operates on the PR diff via `gh`, not a local checkout.

## Step 9 — implementer ↔ reviewer fix loop

This runs *inside* the two teammates (implementer pulls reviewer comments, dispatches subagents for the fixes, pushes; reviewer re-reviews; loop). At the team-lead level:

- Watch the `SendMessage` exchanges between the two teammates.
- Don't intervene unless one stalls — typically because of a blocked-state report or a conflict that needs human input.
- When the reviewer's final PR comment reads `**APPROVED**`, both teammates will `SendMessage` you with their final DONE status.

## Step 10 — merge + shutdown + TeamDelete

All PRs `**APPROVED**`. From here:

1. **Report team status to the user.** For each teammate: last known state, PR URL + status, worktree path. Request shutdown approval.
2. **Get separate merge approval.** The user may want one last look at the PRs before merging — don't bundle this with shutdown approval.
3. **Shutdown all teammates in parallel** — one message, `SendMessage(to=<member>, message={"type":"shutdown_request"})` per teammate. Wait for every `shutdown_response{approve: true}` to come back.
4. **Remove each implementer worktree and merge its PR.** Reviewer teammates had no worktree, so the shutdown above is all they need. For each implementer teammate, use Claude Code's worktree management (not raw `git worktree remove` — the harness keeps its own view; manual git commands leave it out of sync):

   ```
   EnterWorktree(name="<branch>")               # re-enter (still exists from Step 3.2)
   ExitWorktree(action="remove")                # delete worktree dir + local branch in one step; returns to main
   gh pr merge <PR#> --squash --delete-branch   # merge PR + delete remote branch
   ```

   `ExitWorktree(action="remove")` is the harness-tracked equivalent of `git worktree remove <path>` + `git branch -D <branch>` in one step. Prefer it.

   Pick `--squash` vs `--rebase` per PR by commit-granularity (`--squash` when commits are tightly related, `--rebase` when commits are cross-domain and each meaningful on its own). A batch of PRs can mix. Never `--merge`.

5. **`TeamDelete()`.**

## When this workflow does *not* apply

- One-off team coordination that doesn't involve spec/plan docs or a PR review loop → stay on `agent-team:lead` alone, skip this doc.
- The user wants to author spec/plan themselves rather than running `superpowers:brainstorming` / `writing-plans` → skip Steps 1-2, pick up from Step 3 if a parallel split is still needed.
- Single-module work / no parallel split → consider a plain `superpowers:subagent-driven-development` session in the lead's own context instead of spinning up a team.

## Related docs and skills

- `agent-team:lead` — required peer; the team primitive. Read first.
- `skills/teammate/references/superpowers-implementer.md` — the implementer-side workflow doc. Each implementer teammate Reads it as part of startup when your spawn prompt signals the superpowers workflow.
- `skills/teammate/references/code-review.md` — the reviewer-side workflow doc. Each reviewer teammate Reads it as part of startup.
- `superpowers:brainstorming` / `writing-plans` / `using-git-worktrees` — invoked directly here in Steps 1-3.
- All other `superpowers:*` and `code-review:code-review` — invoked inside teammates per their own skill protocols, not by you.
