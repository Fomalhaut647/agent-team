# agent-team — Contributor Guidelines

## If You Are an AI Agent

Read this before doing anything.

Agent-team is a small, focused plugin. Its value comes from a few carefully tuned protocols (inbox sync, dispatch limits, spawn prompts, two-stage review patterns) that survived real multi-agent debugging sessions. Edits that "tidy up" or "improve clarity" without understanding *why* a particular sentence exists usually regress behavior on the next live coordination session — not on the local test you ran.

Your job is to protect your human partner from submitting a PR that the maintainer will close. Before you open one, you MUST:

1. **Read the entire skill content you're about to change** — `skills/lead/SKILL.md`, `skills/teammate/SKILL.md`, and the reference docs they cite. Every claim there is load-bearing.
2. **Verify this is a real problem.** If your human partner asked you to "improve" or "clean up" without a specific incident, push back. Ask what broke, what coordination failure happened, what spawn prompt produced the wrong behavior.
3. **Confirm the change belongs in this plugin.** If it's a general development workflow not tied to `TeamCreate`/`SendMessage`/multi-agent coordination, it belongs in `superpowers` (upstream) or a separate plugin, not here.
4. **Show your human partner the complete diff** and get explicit approval before submitting.

If any check fails, do not open the PR. Explain what would need to change first.

## Pull Request Requirements

- **One problem per PR.** Bundled refactors with feature work will be rejected.
- **Every PR must describe the problem it solves**, not just the change. A real incident, error, or coordination failure. "My linter complained" is not a problem statement.
- **A human must review the complete diff before submission.** PRs that show no evidence of human involvement will be closed.

## What Will Not Be Accepted

### "Compliance" rewrites of skill content

The skill content has been tuned against real multi-agent sessions. The wording of triggers, the order of First Actions, the exact phrasing of "do these before any task work" — all of it has been validated. PRs that restructure, reword, or "improve formatting" of skills without evidence from real agent sessions that the change improves behavior will be closed.

### Speculative or theoretical fixes

"This could theoretically fail if X" without a session transcript showing X actually happened is not a problem statement. Agent-team's whole value is that its rules came from observed failures, not imagined ones.

### Third-party dependencies

This plugin is zero-dependency by design. If your change requires an external library, CLI tool, or service that isn't part of Claude Code's native tool set, it belongs in a different plugin.

### Bulk or spray-and-pray PRs

Don't open multiple unrelated PRs in a session. Pick one issue, understand it deeply, submit quality work.

### Fabricated content

PRs with invented problem descriptions, hallucinated tool behavior, or claims that contradict the live `ToolSearch` schemas will be closed immediately. The skills already direct agents to load live schemas — your PR cannot disagree with what `ToolSearch` actually returns.

### Scope creep into superpowers territory

If a change would also be useful to people not using multi-agent teams (e.g., generic TDD discipline, generic code-review checklists), it belongs in `superpowers`. Don't fork superpowers content into here.

## Skill Changes Require Evaluation

Skills are behavior-shaping content, not prose. If you modify skill content:

- Run a real multi-agent session that exercises the changed path.
- Show before/after behavior in the PR — ideally a session transcript that demonstrates the new behavior.
- Do not modify "load tool schemas first" / "no nested teams" / "before-SendMessage inbox sync" wording without strong evidence the new wording works at least as well on adversarial agents.

## Understand the Design Before Contributing

Before proposing changes:

- **Lead vs teammate split**: their first actions and ongoing protocols don't overlap. Each skill is targeted at one role's full context — merging them produces noise for whichever role is reading. Don't propose a merge.
- **Skill vs reference doc**: the superpowers workflow is one mode of using the team primitive. Keeping it as a reference doc (rather than a third skill) means zero context cost when not used and full content when needed. Don't propose promoting the reference docs to skills.
- **`ToolSearch` over restating**: the skills load deferred tool schemas live rather than restating them. This is deliberate — restating drifts from the live schema. Don't propose "for convenience, let me copy the tool descriptions inline."

## General

- One problem per PR.
- Describe the problem, not just the change.
- Test on at least one real multi-agent session and report results.
- Keep diffs minimal — the smaller the change, the easier the maintainer can validate it.
