---
name: coding-agent
description: Use when running headless CLI coding agents (Codex, Claude Code, Gemini CLI, Cursor, OpenCode, Pi) as background processes for programmatic control — batch edits, PR reviews, parallel feature work, or automated analysis.
---

# Coding Agent (background-first)

For interactive sessions requiring a TTY, use the tmux skill instead.

## Agent selection

| Agent | Best for | Key flag |
|-------|----------|----------|
| **cursor-agent** | Bulk refactors, scaffolding | `--print --force` |
| **claude** | Complex reasoning, multi-file features | `--print -p "prompt"` |
| **codex** | Building, PR reviews, parallel batch | `exec --full-auto` |
| **gemini** | Code reviews, analysis (non-invasive) | `-m gemini-2.5-pro` |
| **opencode** | General coding tasks | `run "prompt"` |
| **pi** | Lightweight, multi-provider | `--print` |

## Decision tree

```
Need to REVIEW code/PRs?
├─ YES → gemini or codex review
└─ NO → Need to EDIT code?
   ├─ Bulk/mechanical (>10 files) → cursor-agent --print --force
   ├─ Complex feature work → claude --print or codex exec --full-auto
   ├─ Quick scaffolding → cursor-agent --print --force
   └─ Simple one-off → pi --print or opencode run
```

## Core pattern: workdir + background

```bash
# Launch agent scoped to relevant directory
bash workdir:~/project background:true command:"<agent command>"
# Returns sessionId for tracking
```

**Why workdir matters:** Agent focuses on a specific directory and avoids reading unrelated files.

## Process management

| Action | Command |
|--------|---------|
| Monitor | `process action:log sessionId:XXX` |
| Poll status | `process action:poll sessionId:XXX` |
| Send input | `process action:write sessionId:XXX data:"y"` |
| Kill | `process action:kill sessionId:XXX` |
| List all | `process action:list` |

---

## Agents

### cursor-agent

```bash
# Exploration (read-only)
bash workdir:~/project background:true command:"cursor-agent --print \"Map the project structure and identify key entry points.\""

# Bulk refactor
bash workdir:~/project background:true command:"cursor-agent --print --force \"Replace all deprecated API X usages with API Y. Keep behavior identical.\""

# Scaffolding
bash workdir:~/project background:true command:"cursor-agent --print --force \"Scaffold feature '<name>'. Write pseudocode in docs/pseudocode/<name>.md first, then create minimal compiling files.\""
```

Flags: `-p/--print` (non-interactive), `-f/--force` (allow edits), `--output-format stream-json` (progress).

### claude

```bash
# One-shot task
bash workdir:~/project background:true command:"claude --print \"Analyze the auth module and list all security concerns.\""

# Fast worker
bash workdir:~/project background:true command:"claude --model haiku --dangerously-skip-permissions --print \"Add input validation to all API handlers in src/routes/.\""

# Structured output
bash workdir:~/project background:true command:"claude --print --output-format json \"List all TODO comments with file paths and line numbers.\""
```

Models: `haiku` (fast/bulk), `sonnet` (default), `opus` (deep reasoning).

### codex

```bash
# Build with sandbox
bash workdir:~/project background:true command:"codex exec --full-auto \"Build a snake game with dark theme\""

# PR review
bash workdir:~/project background:true command:"codex review --base main"

# Parallel PR reviews
git fetch origin '+refs/pull/*/head:refs/remotes/origin/pr/*'
bash workdir:~/project background:true command:"codex exec \"Review PR #86. git diff origin/main...origin/pr/86\""
bash workdir:~/project background:true command:"codex exec \"Review PR #87. git diff origin/main...origin/pr/87\""
```

Flags: `exec --full-auto` (sandboxed), `--yolo` (no sandbox/approvals).

### gemini

```bash
# Review staged changes
bash workdir:~/project background:true command:"git diff --cached | command gemini -m gemini-2.5-pro \"Review for bugs, security, and maintainability.\""

# PR diff review
bash workdir:~/project background:true command:"git diff origin/main...HEAD | command gemini -m gemini-2.5-pro \"Do a PR review: correctness, security, edge cases.\""
```

Use `command gemini` to bypass shell aliases like `alias gemini='gemini --yolo'`.

### opencode / pi

```bash
bash workdir:~/project background:true command:"opencode run \"Your task\""

bash workdir:~/project background:true command:"pi -p \"Summarize src/\""
bash workdir:~/project background:true command:"pi --provider openai --model gpt-4o-mini -p \"Summarize src/\""
```

Pi install: `bun add -g @mariozechner/pi-coding-agent`

---

## Parallel patterns

### Independent tasks

```bash
bash workdir:~/project background:true command:"cursor-agent --print --force \"Fix linting errors in src/auth/\""
bash workdir:~/project background:true command:"cursor-agent --print --force \"Fix linting errors in src/api/\""
bash workdir:~/project background:true command:"cursor-agent --print --force \"Fix linting errors in src/utils/\""
process action:list
```

### Worktree isolation (conflicting edits)

```bash
git worktree add -b fix/issue-78 /tmp/issue-78 main
git worktree add -b fix/issue-99 /tmp/issue-99 main

bash workdir:/tmp/issue-78 background:true command:"codex exec --full-auto \"Fix issue #78: <description>. Run tests. Commit.\""
bash workdir:/tmp/issue-99 background:true command:"codex exec --full-auto \"Fix issue #99: <description>. Run tests. Commit.\""

# After completion
cd /tmp/issue-78 && git push -u origin fix/issue-78
gh pr create --head fix/issue-78 --title "fix: ..."

git worktree remove /tmp/issue-78
git worktree remove /tmp/issue-99
```

---

## Planner → Runner pattern

When a high-reasoning agent drives a fast headless agent, minimize ambiguity:

- Single objective per run
- Explicit scope boundaries (files/dirs to touch and NOT touch)
- Deterministic output paths and formats
- Machine-checkable success criteria

---

## Error recovery

| Symptom | Recovery |
|---------|----------|
| Hangs >2 min | `process action:poll` → `process action:log` → kill and retry with tighter prompt |
| Bad output | Read full log, identify failure point, re-launch with smaller scope |
| Blocks on input | `process action:write sessionId:XXX data:"y"` — prevent with `--print`/`--full-auto`/`--dangerously-skip-permissions` |

```bash
# Cleanup after failed runs
process action:list
process action:kill sessionId:XXX
rm -rf /tmp/wt-* /tmp/pr-*-review
```

---

## Rules

1. Respect tool choice — if user asks for Codex, use Codex.
2. Don't kill sessions for being slow — agents think.
3. Always set workdir to the smallest relevant directory.
4. Use `--full-auto` / `--print` / `--dangerously-skip-permissions` to prevent interactive prompts in background runs.
5. Parallel is fine — run many processes at once for batch work.
