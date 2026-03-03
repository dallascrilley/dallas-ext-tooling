# dallas-ext-tooling

> L1 extension plugin providing implementation skills for building reusable CLIs, MCP servers, pipelines, and task runners. Pairs with the L0 `building-reusable-tools` skill in dallas-base-plugin.

## Structure

- `commands/` — Slash commands
  - `build-tool.md` — Build a new reusable tool and register it in the tool index
  - `tool-index.md` — Manage per-project tool index and global tool catalog
- `hooks/` — Event hooks
  - `skill-rules.json` — Skill discovery triggers for tooling skills
- `skills/` — Implementation skills for tool building
  - `create-cli/` — CLI design and UX (language-agnostic surface area)
  - `dc-cli-kit/` — TypeScript CLI implementation with dc-cli-kit framework
  - `mcporter/` — MCP server interaction and codegen via mcporter CLI
  - `pipeline/` — Multi-step workflow orchestration
  - `justfile-designer/` — Task runner and justfile generation

## Prerequisites

- `mcporter` CLI for MCP server skills
- `just` CLI for justfile skills
- `dc-cli-kit` npm package for TypeScript CLI skills
