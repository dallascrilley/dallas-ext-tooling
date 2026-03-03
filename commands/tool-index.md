# /tool-index

Manage the project's tool index and global tool catalog. Helps agents discover previously built tools before building new ones.

## Usage

```
/tool-index [subcommand] [args]
```

If no subcommand is given, default to `list`.

## Subcommands

### list (default)

Read both the per-project index and global catalog, then display them as Markdown tables.

**Per-project index:** `.claude/tool-index.json` in the current project root.
**Global catalog:** `~/.claude/tool-catalog.json`.

**Steps:**

1. Read `.claude/tool-index.json`. If it doesn't exist, show "No project tool index found. Run `/tool-index add` to create one."
2. If it exists, parse JSON. Wrap in try/catch — if malformed, show: "Your tool-index.json contains invalid JSON at .claude/tool-index.json" and stop.
3. Display project tools as a Markdown table:
   ```
   ## Project Tools
   | Name | Type | Location | Purpose | Usage |
   |------|------|----------|---------|-------|
   ```
4. Read `~/.claude/tool-catalog.json`. If it doesn't exist, show "No global tool catalog found."
5. If it exists, parse JSON with same error handling.
6. Display global tools as a Markdown table:
   ```
   ## Global Tool Catalog
   | Name | Type | Project | Purpose | Usage |
   |------|------|---------|---------|-------|
   ```

### add

Interactively register a new tool to the per-project index.

**Steps:**

1. Ask the user for each field:
   - **name** (required): Tool name (kebab-case)
   - **type** (required): One of `cli`, `mcp-server`, `justfile-recipe`, `pipeline`, `shell-script`
   - **location** (required): Relative path or npm package name
   - **purpose** (required): One-line description
   - **usage** (required): Example invocation (e.g., `tool-name --help`)
   - **tags** (optional): Comma-separated searchable tags
2. Ensure `.claude/` directory exists (`mkdir -p .claude`)
3. Read `.claude/tool-index.json` or initialize with `{"tools":[]}`
   - If JSON is malformed, warn the user and offer to reset
4. Check for duplicate by `name`. If found, ask whether to update or skip.
5. Create the entry with `created` set to today's date (`YYYY-MM-DD`)
6. Append to the `tools` array and write back
7. Suggest: "Run `/tool-index sync` to add this to your global catalog."

### remove `<name>`

Remove a tool entry by name from the per-project index.

**Steps:**

1. Read `.claude/tool-index.json`. If missing, show "No project tool index found."
2. Parse JSON with try/catch error handling.
3. Find the entry matching `name`. If not found, show "No tool named '<name>' found in the project index."
4. Remove the entry from the `tools` array.
5. Write back the updated JSON.
6. Confirm: "Removed '<name>' from the project tool index."
7. Note: "This does not remove the tool from the global catalog. Edit `~/.claude/tool-catalog.json` manually if needed."

### search `<keyword>`

Search for tools by name, type, purpose, or tags across both sources.

**Steps:**

1. Read both `.claude/tool-index.json` and `~/.claude/tool-catalog.json` (skip missing files silently).
2. Parse each with try/catch. If malformed, warn but continue with the other source.
3. Filter tools where `keyword` appears (case-insensitive) in any of: `name`, `type`, `purpose`, `tags` array entries.
4. Display matching results as a Markdown table with a `Source` column indicating `project` or `global`:
   ```
   ## Search Results for "<keyword>"
   | Name | Type | Source | Purpose | Usage |
   |------|------|--------|---------|-------|
   ```
5. If no matches, show "No tools found matching '<keyword>'."

### sync

Copy per-project tool entries to the global catalog, resolving paths and deduplicating.

**Steps:**

1. Read `.claude/tool-index.json`. If missing, show "No project tool index to sync."
2. Parse with try/catch error handling.
3. Determine the absolute project root path (use `pwd` or equivalent).
4. Read `~/.claude/tool-catalog.json` or initialize with `{"tools":[]}`.
   - If the file doesn't exist, create it. Ensure `~/.claude/` directory exists (`mkdir -p ~/.claude`).
   - Parse with try/catch error handling.
5. For each project tool entry:
   - **Resolve location:** If `location` starts with `.` or `/`, resolve to absolute path using the project root. If it contains no path separators (package name), keep as-is.
   - **Set project field:** Add `project` with the absolute project root path.
   - **Deduplicate:** If a tool with the same `name` already exists in the global catalog, update it in place. Otherwise, append.
6. Write back `~/.claude/tool-catalog.json`.
7. Report: "Synced N tool(s) to global catalog at ~/.claude/tool-catalog.json."

## Error Handling

- **Missing files/directories:** Create on first access, never error on fresh state.
- **Malformed JSON:** Wrap all `JSON.parse` calls in try/catch. Report clear error: "Your [filename] contains invalid JSON at [path]". Do not crash or silently ignore.
- **Missing fields:** When adding, require `name`, `type`, `location`, `purpose`, and `usage`. Tags are optional (default to empty array).

## JSON Schemas

**Per-project (`.claude/tool-index.json`):**
```json
{
  "tools": [
    {
      "name": "tool-name",
      "type": "cli | mcp-server | justfile-recipe | pipeline | shell-script",
      "location": "relative/path or npm package name",
      "purpose": "One-line description",
      "usage": "tool-name --help",
      "created": "YYYY-MM-DD",
      "tags": ["optional", "searchable", "tags"]
    }
  ]
}
```

**Global catalog (`~/.claude/tool-catalog.json`):**
```json
{
  "tools": [
    {
      "name": "tool-name",
      "type": "cli",
      "location": "/absolute/path/to/tool or npm package name",
      "project": "/absolute/path/to/project-root",
      "purpose": "One-line description",
      "usage": "tool-name --help",
      "created": "YYYY-MM-DD",
      "tags": ["tag1"]
    }
  ]
}
```
