---
name: pipeline
description: Chain multiple operations together in pipelines. Use for multi-step workflows, combining research with analysis, and complex automated tasks.
---

# Pipeline Orchestration

Chain multiple tools and operations together in multi-step workflows.

## Pipeline Patterns

### Transform Chain

```bash
#!/bin/bash
# transform.sh - Chain of transformations

INPUT=$1

# Step 1: Extract
EXTRACTED=$(echo "$INPUT" | process_step_1)

# Step 2: Structure
STRUCTURED=$(echo "$EXTRACTED" | process_step_2)

# Step 3: Validate
VALIDATED=$(echo "$STRUCTURED" | process_step_3)

echo "$VALIDATED"
```

### Conditional Pipeline

```bash
#!/bin/bash
# conditional.sh - Branch based on analysis

INPUT=$1

# Analyze type
TYPE=$(analyze_input "$INPUT")

case $TYPE in
  bug*)
    handle_bug "$INPUT"
    ;;
  feature*)
    handle_feature "$INPUT"
    ;;
  question*)
    handle_question "$INPUT"
    ;;
esac
```

### Parallel Pipeline

```bash
#!/bin/bash
# parallel.sh - Run analysis in parallel

INPUT=$1

# Run in parallel
echo "$INPUT" | analyze_technical > /tmp/technical.txt &
echo "$INPUT" | analyze_business > /tmp/business.txt &
echo "$INPUT" | analyze_risk > /tmp/risk.txt &
wait

# Combine results
combine_analyses /tmp/technical.txt /tmp/business.txt /tmp/risk.txt
```

## Common Pipeline Templates

### Code Review Pipeline

```bash
#!/bin/bash
# code-review.sh FILE

FILE=$1
CODE=$(cat "$FILE")

# Step 1: Static analysis
echo "=== Linting ===" > /tmp/review.txt
npx eslint "$FILE" 2>&1 >> /tmp/review.txt

# Step 2: Type check
echo "" >> /tmp/review.txt
echo "=== Type Check ===" >> /tmp/review.txt
npx tsc --noEmit "$FILE" 2>&1 >> /tmp/review.txt

# Step 3: AI review (use your preferred CLI)
echo "" >> /tmp/review.txt
echo "=== AI Review ===" >> /tmp/review.txt
# Pipe code to your AI CLI of choice for review
cat /tmp/review.txt
```

### Research Pipeline

```bash
#!/bin/bash
# research.sh TOPIC

TOPIC=$1

# Step 1: Initial research
echo "Researching: $TOPIC"
INITIAL=$(research_tool "$TOPIC")

# Step 2: Find gaps
GAPS=$(echo "$INITIAL" | identify_gaps)

# Step 3: Fill gaps
FOLLOWUP=$(echo "$GAPS" | research_followup "$TOPIC")

# Step 4: Synthesize
synthesize "$INITIAL" "$FOLLOWUP"
```

## Error Handling

### With Retry

```bash
#!/bin/bash
# retry-pipeline.sh

retry() {
  local n=1
  local max=3
  local delay=2
  while true; do
    "$@" && return 0
    if [[ $n -lt $max ]]; then
      ((n++))
      echo "Retry $n/$max in ${delay}s..."
      sleep $delay
    else
      return 1
    fi
  done
}

# Use in pipeline
retry your_command "args"
```

### With Fallback

```bash
#!/bin/bash
# fallback-pipeline.sh

# Try primary tool, fallback to secondary
result=$(primary_tool "input" 2>/dev/null) || \
result=$(fallback_tool "input")

echo "$result"
```

## Best Practices

1. **Save intermediate results** — Debug easier
2. **Add timeouts** — Prevent hanging
3. **Handle errors** — Check return codes
4. **Log progress** — Track long pipelines
5. **Test incrementally** — Verify each step
6. **Use temp files** — For complex data passing
7. **Clean up** — Remove temp files after
8. **Use `set -euo pipefail`** — Fail fast on errors
