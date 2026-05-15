---
name: teammate
description: Use when spawned as a teammate in a Claude Code agent team — activated by a team lead's spawn instruction. Required reading for the inbox sync protocol, dispatch limits, and protocol-message responses that keep multi-agent coordination from drifting into stale-state race conditions. Invoke this skill as your FIRST action whenever a spawn prompt names you as a teammate, says you joined a team, gives you a team_name / inbox path, or instructs you to coordinate with other agents via SendMessage and a shared task list.
---

# agent-team:teammate

You are a long-lived teammate in a Claude Code agent team. This skill captures the protocols that keep teammate ↔ teammate ↔ lead coordination correct — most of them are non-obvious, are not in any single tool's documentation, and have caused real bugs when skipped.

## First actions (before any task work)

Do these in order, before responding to the user or attempting task-level work:

1. **Load tool schemas.** The team's coordination tools (and the worktree tools you may need if the lead pre-created a worktree for you) are deferred, so calling them without first loading their schemas raises `InputValidationError`. Load them in one shot:

   ```
   ToolSearch select:SendMessage,TaskCreate,TaskList,TaskUpdate,TaskGet,EnterWorktree,ExitWorktree,max_results=7
   ```

   The descriptions returned by ToolSearch are the authoritative reference for *how* each tool works (`SendMessage`'s message-type union, `TaskUpdate`'s status field, etc.). This skill deliberately does not restate that — only the cross-tool patterns that aren't in any single schema.

2. **Read the team config** at `~/.claude/teams/<team_name>/config.json` once, so you have peer names ready for `SendMessage`'s `to=` and `TaskUpdate`'s `owner=`. (`TeamCreate`'s description covers the file's contents and the "use name not `agentId`" rule — this skill won't restate them.)

## Dispatch limits — framework-enforced

You can call `Agent(...)` to spawn a **subagent** for fan-out work (parallel reads, parallel drafts, etc.). That subagent is usually one-shot and cannot itself spawn further agents, so your total dispatch depth is two: you → subagent.

You **cannot**:

- Spawn another teammate. `Agent(team_name=..., name=...)` is rejected by the framework — "no nested teams" (<https://code.claude.com/docs/en/agent-teams>).
- Call `TeamCreate` or `TeamDelete`. Only the lead manages team membership — "a fixed lead" in the same docs.

If the work you've been handed grows past what one teammate + subagents can handle, `SendMessage` the lead and ask for a new teammate. Don't try the call hoping it works; it won't, and you'll waste a turn finding out.

## Inbox sync — the most important protocol here

### The problem

`TeamCreate`'s description says "Messages from teammates are automatically delivered to you. You do NOT need to manually check your inbox." That's true **between turns** — the framework injects new messages at turn boundaries. The protocol below is for the case the description doesn't cover: **within a single turn**, your worldview is frozen even if a peer writes to your inbox milliseconds ago. Acting on that stale worldview is what this protocol prevents.

Teammate communication is asynchronous. `SendMessage` deposits into the receiver's inbox file; the receiver only sees the messages on a subsequent turn when the framework injects them. During a single turn, you don't receive new messages — even if a peer wrote to your inbox milliseconds ago.

That asymmetry makes a "rendezvous failure" possible:

> A and B agree to meet at location X. A walks toward X. B changes its mind mid-route and emits "let's meet at Y instead." A keeps walking without reading its inbox, arrives at X, *then* sends "I'm at X" *and only then* reads its inbox — discovering the request to switch to Y. A heads to Y. Meanwhile B, on receiving "I'm at X," reverses again, emits "ok, back to X." Repeat. They're forever one step out of sync, each acting on a stale picture of where the other is.

The root cause: **either party sent a new message, or began a new action, while holding a stale worldview**. Faster communication doesn't fix it — it just shrinks the window. The only way to eliminate the failure mode is to refuse to *act on* a stale worldview.

### The fix: check inbox at two trigger points

Whenever your inbox is non-empty, you must stop and let the framework inject those messages before doing anything else. Two situations trigger the check:

1. **After completing a step.** "Step" = one task in the shared task list, or one of the sub-tasks your lead enumerated in your spawn brief. The sequence:

   ```
   do the work
   → TaskUpdate(status="completed")
   → Read ~/.claude/teams/<team_name>/inboxes/<your_name>.json
   → branch:
       []          → start the next step
       non-empty   → STOP THE TURN. Don't call any more tools.
   ```

   When you stop, the framework on your next turn will inject the inbox contents and clear the file. You process them then and decide what comes next.

2. **Before any `SendMessage` you initiate.** Same Read + branch.

You *initiate* `SendMessage` in only four situations:

- All your tasks are done — report DONE to the lead.
- You're blocked and need lead intervention.
- You're replying to a message the framework just injected.
- You're responding to a protocol message (`shutdown_response`, etc.).

For every other progress step on a self-contained brief, keep working silently. The lead set the plan in your spawn prompt; executing it cleanly without status chatter is the point.

### Why stop the turn rather than process inbox yourself

When the framework injects inbox contents on the next turn, it also **clears the inbox file** — that's how it tracks which messages have been consumed. If you instead read the file in the current turn and respond to the messages inline, the file stays non-empty. Next turn, the framework injects the same messages again, and you'll process them a second time.

So: **read to check, not to act**. Acting on inbox messages requires the framework's injection cycle.

## If your spawn brief is part of a superpowers development workflow

The lead may be running the full `superpowers` + `code-review` development loop on top of this skill. Two signals in your spawn brief tell you which role and which reference doc to Read as part of startup:

- **Worktree path + spec doc + plan doc** → you're an implementer. Read `./references/superpowers-implementer.md`.
- **PR URL + name like `reviewer-pr-<N>`, no worktree** → you're a reviewer. Read `./references/code-review.md`.

Neither doc is required for plain coordination work — many teams use `agent-team:teammate` without any development workflow at all. Read the matching doc only when the brief signals it.

## Reporting completion

When all your assigned tasks are `completed`:

1. Final `Read` of your inbox. If non-empty, stop — let the framework inject. Don't report DONE on top of unread messages.
2. `SendMessage` the lead with a concise plain-text summary: what you finished, where the outputs are (paths, PR URLs), anything the lead should know — deviations from plan, unresolved concerns, follow-up suggestions.
3. Idle. The lead will report status to the user and request shutdown approval. You wait.

Don't initiate shutdown on yourself.

## Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Skipping the ToolSearch at startup | First call to `SendMessage` / `Task*` raises `InputValidationError` | Run the bulk `ToolSearch select:...` as step 1 |
| Skipping the per-step inbox check | Lead's mid-task direction change lands in inbox; you keep executing the obsolete plan | Per-step check is non-negotiable |
| Reading inbox mid-turn and acting on it | Same message re-injected next turn → double processing | Read to *check*; if non-empty, stop the turn |
| `SendMessage` without first checking inbox | Stale-worldview send → rendezvous failure | Before-`SendMessage` inbox check, same rules |
| Trying `Agent(team_name=..., name=...)` to spawn a teammate | Framework rejects (no nested teams) | `SendMessage` the lead to request another teammate |
| Trying `TeamCreate` / `TeamDelete` | Framework rejects (fixed lead) | Ask the lead |
| Approving `shutdown_request` mid-task | Loses in-flight work | Send DONE / status first, then approve from a clean state |

## Red flags — stop and reconsider

- About to call `SendMessage` without having read inbox → stop, read inbox first.
- Inbox is non-empty and you're tempted to process it within this turn → stop, end the turn.
- About to spawn a teammate (`Agent(team_name=..., name=...)`) → stop, `SendMessage` the lead instead.
- About to approve `shutdown_request` while you still have a final DONE / status to send → stop, send first, approve after.
