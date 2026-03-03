# dallas-ext-tooling

Reusable tooling plugin for Claude Code. Provides implementation skills for building CLIs, MCP servers, pipelines, and task runners.

## Skills

| Skill | Purpose |
|-------|---------|
| `create-cli` | Design CLI surface area — args, flags, subcommands, help text, output formats |
| `dc-cli-kit` | Build TypeScript CLIs with dc-cli-kit framework (auth, config, HTTP, errors) |
| `mcporter` | Interact with MCP servers — list, call, auth, codegen |
| `pipeline` | Chain multiple operations in multi-step workflows |
| `justfile-designer` | Generate, enhance, and audit justfiles for any project |

## Installation

```bash
claude plugins install ~/Code/dallas-plugin-marketplace/dallas-ext-tooling
```

## Pairs With

The L0 `building-reusable-tools` skill in `dallas-base-plugin` provides the decision framework for *when* to build a tool. This extension provides the *how*.

## License

MIT
