# Claude Code Plugins

A personal marketplace of Claude Code plugins maintained by [Fomalhaut647](https://github.com/Fomalhaut647).

> **⚠️ Important:** Make sure you trust a plugin before installing, updating, or using it. See each plugin's own README for what it does and how it integrates with Claude Code.

## Installation

Add this marketplace to your Claude Code, then install plugins from it:

```
/plugin marketplace add Fomalhaut647/Claude-Code-Plugins
/plugin install agent-team@claude-code-plugins
```

Or browse via `/plugin > Discover` once the marketplace is added.

## Plugins

| Plugin | Description |
|---|---|
| [`agent-team`](plugins/agent-team) | Coordinate long-running Claude Code agent teams. Two subskills — `agent-team:lead` orchestrates, `agent-team:teammate` runs each worker — plus optional reference docs that layer the `superpowers` + `code-review` development workflow on top. |

## Structure

```
Claude-Code-Plugins/
├── .claude-plugin/
│   └── marketplace.json   # Marketplace index (lists each plugin + its source path)
├── plugins/
│   └── <plugin-name>/     # One self-contained plugin per directory
│       ├── .claude-plugin/plugin.json
│       ├── README.md
│       ├── LICENSE
│       └── skills/ commands/ agents/ ...
└── README.md
```

Each plugin under `plugins/` is a standalone Claude Code plugin and has its own README + LICENSE. `marketplace.json` is just an index that points at them.

## License

Each plugin carries its own LICENSE file. The marketplace metadata in this repository is MIT.
