---
name: ast-grep
description: Use when searching for code patterns structurally (not just text) — async functions missing error handling, calls with specific arguments, code inside certain contexts — using ast-grep's AST-based rules.
---

# ast-grep Code Search

## Workflow

1. Identify the target pattern and language.
2. Create a minimal example snippet that should match.
3. Use `--debug-query=cst` to inspect the AST if `kind` values are unknown.
4. Write the rule (start simple, add constraints one at a time).
5. Test against the snippet before running on the full codebase.
6. Run on the full codebase with `sg scan` or `sg run`.

## CLI Commands

### Inspect AST structure

```bash
# Dump concrete syntax tree to find correct `kind` values
sg run --pattern 'async function $NAME() { $$$BODY }' \
  --lang javascript --debug-query=cst
```

Available formats: `cst`, `ast`, `pattern`.

### Simple pattern search (`sg run`)

```bash
sg run --pattern 'console.log($ARG)' --lang javascript .
sg run --pattern 'function $NAME($$$)' --lang javascript --json .
```

Use for single-node matches without relational logic.

### Rule-based search (`sg scan`)

```bash
# Rule file
sg scan --rule my_rule.yml /path/to/project

# Inline rule (escape $ in shell)
sg scan --inline-rules "id: my-rule
language: javascript
rule:
  pattern: \$PATTERN" /path/to/project

# Test against stdin snippet
echo "const x = await fetch();" | sg scan --inline-rules "id: test
language: javascript
rule:
  pattern: await \$EXPR" --stdin
```

Use for relational rules (`has`, `inside`, `follows`, `precedes`) and composite logic (`all`, `any`, `not`).

## Rule Authoring

### Key rules

- Always add `stopBy: end` to `has`/`inside` relations — without it, traversal stops at the first non-match.
- Use `pattern` for direct matches; use `kind` + relations for structural context.
- Combine constraints with `all: [...]` when evaluation order matters.

### Rule file skeleton

```yaml
id: rule-name
language: javascript
rule:
  all:
    - kind: function_declaration
    - has:
        pattern: await $EXPR
        stopBy: end
    - not:
        has:
          pattern: try { $$$ } catch ($E) { $$$ }
          stopBy: end
```

### Escape metavariables in shell

```bash
# Wrong: shell expands $ARG
sg scan --inline-rules "rule: {pattern: 'console.log($ARG)'}" .

# Correct: escape with \
sg scan --inline-rules "rule: {pattern: 'console.log(\$ARG)'}" .
# Or: single-quote outer, double-quote inner
sg scan --inline-rules 'rule: {pattern: "console.log($ARG)"}' .
```

## Common Patterns

```bash
# Async functions containing await
sg scan --inline-rules "id: async-await
language: javascript
rule:
  all:
    - kind: function_declaration
    - has:
        pattern: await \$EXPR
        stopBy: end" .

# console.log inside class methods only
sg scan --inline-rules "id: console-in-class
language: javascript
rule:
  pattern: console.log(\$\$\$)
  inside:
    kind: method_definition
    stopBy: end" .

# Async functions without try-catch
sg scan --inline-rules "id: async-no-trycatch
language: javascript
rule:
  all:
    - kind: function_declaration
    - has:
        pattern: await \$EXPR
        stopBy: end
    - not:
        has:
          pattern: try { \$\$\$ } catch (\$E) { \$\$\$ }
          stopBy: end" .
```

## Debugging checklist

1. No matches? Simplify the rule — remove sub-rules until something matches.
2. Add `stopBy: end` to any relational rule missing it.
3. Run `--debug-query=cst` to confirm `kind` values.
4. Check metavariable escaping in shell inline rules.
