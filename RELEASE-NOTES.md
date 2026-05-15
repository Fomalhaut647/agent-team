# agent-team Release Notes

## Unreleased

### Repository Restructuring

Migrated from a multi-plugin marketplace layout to a single-plugin layout modeled on `obra/superpowers`. The repo is now itself the plugin rather than a wrapper around one.

- `plugins/agent-team/skills/` → `skills/` (root)
- `plugins/agent-team/LICENSE` → `LICENSE` (root)
- `plugins/agent-team/.claude-plugin/plugin.json` → `.claude-plugin/plugin.json` (root, alongside the existing `marketplace.json`)
- `.claude-plugin/marketplace.json` updated: `source: "./plugins/agent-team"` → `source: "./"`; marketplace name `claude-code-plugins` → `agent-team-dev`
- `plugins/` directory removed entirely
- `README.md` rewritten as a single-plugin README (was a marketplace overview)
- Added `CLAUDE.md` — contributor guide aimed at AI agents working on this plugin
- Added this file

Plugin functionality is unchanged. Skill contents (`agent-team:lead`, `agent-team:teammate`, and their reference docs) are byte-identical to v0.1.0 — only their on-disk paths changed.

## v0.1.0

Initial release.

- `agent-team:lead` — orchestrates a multi-agent team: `TeamCreate`, spawn prompt template, task assignment, hub-and-spoke topology, shutdown sequence, merge + cleanup ordering.
- `agent-team:teammate` — first-action protocol for spawned teammates: load tool schemas, read team config, per-step + before-SendMessage inbox sync, "no nested teams" framework limit, completion reporting.
- Reference docs for layering the `superpowers` development workflow on top: `skills/lead/references/superpowers-workflow.md`, `skills/teammate/references/superpowers-implementer.md`, `skills/teammate/references/code-review.md`.
