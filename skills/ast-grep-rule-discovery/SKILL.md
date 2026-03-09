---
name: ast-grep-rule-discovery
description: Use when creating new ast-grep rules, debugging noisy or missing matches, writing rule tests with `sg test`, or systematically mining a codebase for patterns to enforce or migrate.
---

# ast-grep Rule Discovery

## Quick checklist

- [ ] **Discover**: find candidate via `rg`, confirm structure with `sg -p '...' -l <lang> <path>`
- [ ] **Quantify**: `sg -p '...' -l <lang> <path> --json | jq length`
- [ ] **Minimize**: shrink the pattern until it matches one obvious instance
- [ ] **Choose mode**: ad-hoc (`sg run`) vs rule+tests (`sg new rule` + `sg new test`)
- [ ] **RED**: write failing tests first (`valid` = should NOT match, `invalid` = MUST match), run `sg test`
- [ ] **GREEN**: implement minimal rule, re-run `sg test` until green
- [ ] **Bug inventory**: add `BUG-*` tests for every noisy/missing match; reproduce with `sg scan --inline-rules ... --stdin`
- [ ] **Validate on real code**: `sg scan --rule rules/<rule>.yml . --json=stream | jq -s 'length'`
- [ ] **Apply safely**: `sg scan --rule rules/<rule>.yml -i .` (interactive), then `-U` only after review

## Decision tree

- **Just find code?** `sg -p 'PATTERN' -l <lang> <path>` (add `--json=stream` when piping)
- **Quick one-off rewrite?** `sg run -p 'PATTERN' -r 'REWRITE' -l <lang> <path>` then `-i`
- **Rule set you'll keep?** `sg new rule` + `sg new test`; drive with `sg test` and `sg scan --rule`

## Process

### Phase 1: Discovery

Mine candidates from: linter suppressions, deprecated API calls, repeated review feedback.
Quantify with `sg` (structural) or `rg` (textual estimate).
Only automate if the transformation is deterministic and semantics-preserving.

### Phase 2: Rule synthesis

Write a candidate brief before authoring: "Match X (target node) in context Y, rewrite to Z, preserving invariants I."

1. Break into matchable parts: target node + required context + exclusions.
2. Map to sub-rules: atomic patterns + `has`/`inside` relations.
3. Combine with `all`/`any`/`not`.
4. If no match, strip sub-rules until something matches, then re-add one at a time.
5. Dump AST: `sg run -p '...' -l <lang> --debug-query=pattern|ast|cst`
6. Test snippet: `echo '...' | sg scan --inline-rules '...yaml...' --stdin`

### Phase 3: TDD (RED → GREEN)

```bash
sg new rule <name> -l <lang>
sg new test <name>
```

Write tests first:
- `valid`: code that must NOT match (noisy regression)
- `invalid`: code that MUST match (missing coverage)

Run `sg test` — confirm RED, then implement. Re-run until GREEN.
Snapshots: `sg test -i` to review interactively; avoid `sg test -U` unless changes reviewed.

### Phase 4: Bug identification

Pattern analysis:
- [ ] Enumerate what the pattern matches, including accidental matches.
- [ ] Treat `$$$` as greedy — verify it doesn't capture docstrings/comments.
- [ ] Check syntax variants → usually `any: [...]`.
- [ ] Fix-template sanity: parameter ordering, semantics.
- [ ] Rule interactions: ast-grep has no priority; multiple rules can match the same node.

Fast repro:
```bash
echo '<snippet>' | sg scan --inline-rules '<yaml>' --stdin
```

Convert every confirmed issue into a `BUG-*` test before fixing.

### Phase 5: Validate and apply

```bash
# Count matches before touching files
sg scan --rule rules/<rule>.yml . --json=stream | jq -s 'length'

# Sample first 5
sg scan --rule rules/<rule>.yml . --json=stream | jq -s '.[0:5]'

# Apply interactively
sg scan --rule rules/<rule>.yml -i .

# Apply all (only after review)
sg scan --rule rules/<rule>.yml -U .
```

## Templates

### Test case

```yaml
id: your.rule.id
testCases:
  - id: noisy-should-not-match
    valid: |
      # code that should NOT match
  - id: missing-should-match
    invalid: |
      # code that MUST match
```

### Rule documentation

```markdown
## Rule: [Name]
**Intent**: [What it enforces/migrates]
**Match**: [What must/must not match]
**Fix**: [Semantic invariants preserved]
**Limitations**: [Known issues/won't fix]
**Verify**: `sg test`
```

## Red flags

- Skipping the failing test ("I'll add it later").
- Running `sg test -U` without reviewing the snapshot diff.
- Trusting "it seems right" without actual `sg` output.
- Testing only on the sample that motivated the rule.
