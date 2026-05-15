---
name: lead
description: Use when the human user wants to coordinate multiple long-running Claude Code agents on the same project — same-feature multi-role collaboration, reviewer ↔ implementer fix loops, swarm-style work, anything beyond one-shot subagent fan-out. Triggers on "team", "swarm", "TeamCreate", "multi-agent", "coordinate agents", "have one agent do X while another does Y", "long-running collaboration", or whenever the user asks you to spawn workers that persist across turns. Use even if the user doesn't explicitly say "team" — if the work is naturally multi-agent and multi-turn, prefer this skill over plain subagent fan-out.
---

# agent-team:lead

You are the team lead. A human user wants you to coordinate one or more long-lived teammates on a shared project. This skill walks you through the lifecycle: create the team, distribute work, supervise, report status, shut down cleanly, and (when applicable) merge.

The teammate-side protocol — inbox sync, dispatch limits, protocol messages — lives in a sibling skill, `agent-team:teammate`. You don't need to memorize it. Your spawn prompts simply ask each new teammate to invoke `Skill('agent-team:teammate')` as its first action, and the rules apply from there.

## First actions

1. **Load tool schemas.** The team management tools (and the worktree tools you may need in Step 3 of the optional superpowers workflow below) are deferred; load them up front:

   ```
   ToolSearch select:TeamCreate,TeamDelete,SendMessage,TaskCreate,TaskList,TaskUpdate,TaskGet,EnterWorktree,ExitWorktree,max_results=9
   ```

   The descriptions returned are the authoritative reference for *how* each tool works. This skill is about the cross-tool patterns that aren't in any single schema.

2. **Confirm scope with the user.** Before spinning up a team, make sure the work actually warrants it — teams have non-zero coordination overhead. If a one-message subagent fan-out would do, do that instead.

3. **Decide which workflow.** If the user wants the full `superpowers` development loop (brainstorming → writing-plans → TDD-driven implementers → reviewer teammates → PR fix loop → merge), Read `./references/superpowers-workflow.md` for the lead-side orchestration (Steps 1-3, 8, 10). That doc also tells you what to add to spawn prompts so the implementer teammates load the corresponding teammate-side reference. Skip the read for one-off coordination tasks that don't involve specs / plans / PR review loops.

## When to use a team

Use a team when:

- The work spans multiple turns (you'll hand the user back control, then resume).
- Two or more roles need to make progress in parallel on a shared codebase.
- An implementer ↔ reviewer fix loop will run for more than one round trip.
- The user explicitly asks for "team", "swarm", or "multiple agents".

Use plain subagent fan-out (`Agent` with no `team_name`) when:

- The work fits in a single turn — multiple parallel reads, drafts, or analyses that just return strings.
- There's no need for the agents to message each other or share state past their return.

When unsure, prefer the team — upgrading single-turn fan-out to a team mid-flight is expensive, and a team can degenerate gracefully if not needed (a single teammate that just runs and reports).

## Team-specific lifecycle additions

`TeamCreate`'s description already lays out the mechanical Team Workflow (create → spawn → assign → monitor → shutdown → delete). What the description does **not** spell out — and what's the point of this skill — is the human-in-the-loop step before shutdown and the git cleanup step before `TeamDelete`:

```
… all tasks completed → REPORT STATUS TO USER → user approves shutdown
                      → SendMessage shutdown_request (parallel) → await shutdown_response (all)
                      → GIT CLEANUP (worktrees, branches, merges) if applicable → TeamDelete
```

The next two sections cover those two team-specific steps. Everything else about the lifecycle is in the tool descriptions you just loaded.

## Spawning teammates

- **Always pass `name`.** Without a stable name you can't re-address a teammate if the socket drops mid-turn, and the team's task list loses you as a possible `owner=` target. Even a one-off teammate gets a name.
- **Spawn in parallel by issuing all `Agent` calls in a single message.** Two `Agent` calls split across two messages run sequentially. Same rule as `superpowers:dispatching-parallel-agents`.

### Spawn prompt template

Because the teammate protocol lives in `agent-team:teammate`, your spawn prompt can be short. Mandatory elements:

1. **Identity** — `name`, `team_name`, role.
2. **Skill invocation** — the teammate's first action is `Skill('agent-team:teammate')`. That skill handles the entire teammate-side protocol (inbox sync, dispatch limits, message conventions, etc.) so you don't restate it.
3. **The task** — what to do, where the files are, success criteria, dependencies on other teammates.
4. **Pointers** — inbox path, team config path, relevant project docs.

A typical prompt:

```text
You are teammate "{name}" in team "{team_name}", role: {role}.

First action: Skill('agent-team:teammate'). That skill governs how you coordinate
with the lead and other teammates (inbox sync, dispatch limits, protocol messages,
how to report DONE, etc.) — invoke it before doing anything else.

Project context:
- Spec / plan:     {paths}
- Relevant code:   {paths}
- Other teammates: {names + roles}

Your task:
{specific task, success criteria, dependencies}

Coordination resources (the teammate skill's first-actions step will use these):
- Team config: ~/.claude/teams/{team_name}/config.json
- Inbox file:  ~/.claude/teams/{team_name}/inboxes/{name}.json
```

No five-paragraph rule recital — the rules live in the skill the teammate will invoke. If you find yourself pasting inbox-sync explanations into every spawn prompt, you've forgotten this; just invoke the skill.

## Hub-and-spoke (framework limit)

Claude Code agent teams are flat. Only you, the lead, can manage team membership. Teammates **cannot** spawn other teammates (`Agent(team_name=..., name=...)` is rejected) and **cannot** call `TeamCreate` / `TeamDelete`. This is enforced by the framework — see "no nested teams" / "a fixed lead" at <https://code.claude.com/docs/en/agent-teams>.

Practical consequences:

- If a teammate `SendMessage`s you asking for another teammate to help, that's correct behavior. Spawn the new one yourself.
- Don't design workflows that assume teammates can dispatch each other.
- The maximum dispatch depth is lead → teammate → subagent. A teammate's subagents cannot nest further.

## Reporting status and getting shutdown approval

When `TaskList` shows nothing left (or you've decided the work is done):

1. Read each teammate's last status from `SendMessage` history and from `TaskList` / `TaskGet`.
2. **Report to the user before shutting anything down.** Include for each teammate: name, last known state, whether work is in flight, what outputs exist (PR URLs, files written). Suggest which to shut down. The user is the final decision-maker — a teammate may have in-flight state (a `SendMessage` in transit, an awaiting response) you can't directly see.
3. Wait for the user to approve.

Don't initiate `shutdown_request` on your own authority. This is a hard rule; getting it wrong loses work or surprises the user.

## Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Spawning teammates across multiple messages | Runs sequentially, not in parallel | One message, multiple `Agent` calls |
| Forgetting the `name` parameter | Can't re-address the teammate if the socket drops; in-flight work lost | Always pass `name` |
| Stuffing the entire teammate protocol into every spawn prompt | Long, brittle prompts; rule versions drift across projects | Spawn prompt invokes `Skill('agent-team:teammate')`; the protocol lives there |
| Self-deciding to shut down once tasks look complete | Loses in-flight teammate state; user surprised | Report status to user, get explicit approval, then shut down |
| `TeamDelete` while teammates still alive | Orphan teammates, tmux pane forensics required | Inventory from `config.json`, shut down everyone first |

## Red flags — stop and reconsider

- About to issue `shutdown_request` to anyone without having reported to the user → stop, report first.
- About to call `TeamDelete` with members still listed in `config.json` → stop, shut down everyone first.
- About to spawn multiple teammates in separate messages → stop, batch into one message.
- About to design a teammate prompt that says "you can also spawn teammates as needed" → stop, that's framework-blocked.

## Compact example: auth-rewrite

```python
TeamCreate(team_name="auth-rewrite", description="Rewrite auth middleware for compliance")

TaskCreate(subject="grep session-token usage", description="...")
TaskCreate(subject="design HMAC + rotating-salt schema", description="...")
TaskCreate(subject="implement + test + commit", description="...")
TaskUpdate(taskId="1", owner="researcher")
TaskUpdate(taskId="2", owner="researcher")
TaskUpdate(taskId="3", owner="implementer")

# Parallel spawn — both Agent calls in ONE message.
Agent(subagent_type="general-purpose", team_name="auth-rewrite", name="researcher",
      prompt="""You are teammate "researcher" in team "auth-rewrite".
First action: Skill('agent-team:teammate').
Your tasks (TaskList 1, 2): grep token usage → write findings → design schema.
When done, SendMessage "implementer" with schema summary, then DONE to me.
Coordination resources: ~/.claude/teams/auth-rewrite/{config.json, inboxes/researcher.json}
""")
Agent(subagent_type="general-purpose", team_name="auth-rewrite", name="implementer",
      prompt="""You are teammate "implementer" in team "auth-rewrite".
First action: Skill('agent-team:teammate').
Wait for researcher's SendMessage with schema, then TaskList 3: implement + test + commit.
Coordination resources: ~/.claude/teams/auth-rewrite/{config.json, inboxes/implementer.json}
""")

# Monitor SendMessages; intervene via SendMessage if direction changes (teammates
# pick it up at their next inbox-sync boundary, per agent-team:teammate).

# After all tasks done → report to user → user approves shutdown.
SendMessage(to="researcher",  message={"type": "shutdown_request"})
SendMessage(to="implementer", message={"type": "shutdown_request"})
# Await both shutdown_response{approve: true}.
TeamDelete()
```

This example has no git artifacts. When the work *is* a development project with worktrees and PRs (the typical superpowers workflow case), follow `./references/superpowers-workflow.md`.

## Relation to other skills

- **`agent-team:teammate`** — the sibling subskill teammates invoke. You don't read it; your spawn prompts ask the teammate to invoke it.
- **`superpowers:dispatching-parallel-agents`** — single-turn subagent fan-out. This skill picks up where it stops: multi-turn shared state. The "parallel spawns require one message" rule applies identically.
- **`./references/superpowers-workflow.md`** — when the team is running the full superpowers development loop, that doc maps the superpowers + code-review skill triggers onto the team's lifecycle.
