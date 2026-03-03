---
name: justfile-designer
description: Generates production-quality justfiles by detecting project stack, applying best practices, and matching team conventions. Supports creation, enhancement, and auditing.
license: MIT
metadata:
  version: 1.0.0
---

# Justfile Designer

Generates, enhances, and audits justfiles for any project.

---

## Quick Start

```
justfile-designer                     # Detect stack, generate justfile
justfile-designer: add docker recipes # Add recipes to existing justfile
justfile-designer: audit              # Audit existing justfile for issues
```

---

## Triggers

- `justfile-designer` / `create justfile` / `generate justfile`
- `design justfile for {project}`
- `add recipes to justfile` / `enhance justfile`
- `audit justfile` / `review justfile`
- `justfile for {stack}` (e.g., "justfile for python/fastapi")

| Input | Action |
|-------|--------|
| No justfile exists | **Create** — detect stack, generate full justfile |
| Justfile exists | **Enhance** — analyze gaps, add missing recipes |
| `audit` keyword | **Audit** — check against best practices, report issues |
| Specific recipes requested | **Targeted** — add only what's asked for |

---

## Process

```
USER INPUT
    │
    ▼
┌─────────────────────────────────────────────────┐
│ 1. DETECT MODE (create / enhance / audit)       │
│    • Check if justfile exists                   │
│    • Parse user intent                          │
├─────────────────────────────────────────────────┤
│ 2. STACK DETECTION                              │
│    • Scan project files for stack signals        │
│    • Identify package manager, runtime, tools   │
│    • Check for existing scripts/ directory      │
├─────────────────────────────────────────────────┤
│ 3. RECIPE SELECTION                             │
│    • Match stack to recipe templates             │
│    • Apply user conventions if existing justfile │
│    • Include only relevant categories           │
├─────────────────────────────────────────────────┤
│ 4. GENERATE / ENHANCE / REPORT                  │
│    • Write justfile with proper structure        │
│    • Verify syntax with just --check if avail   │
└─────────────────────────────────────────────────┘
```

---

## Commands

| Command | Action |
|---------|--------|
| `justfile-designer` | Auto-detect and create |
| `justfile-designer: enhance` | Add missing recipes to existing justfile |
| `justfile-designer: audit` | Audit justfile against best practices |
| `justfile-designer: {category}` | Add specific category (e.g., `docker`, `db`) |

---

## Step 1: Detect Mode

```
Check working directory:
├── No justfile found → CREATE mode
├── Justfile found + "audit" requested → AUDIT mode
├── Justfile found + recipes requested → ENHANCE mode
└── Justfile found + no specific request → Ask: enhance or audit?
```

When enhancing, read the existing justfile first to:
- Match indentation style
- Match section divider style
- Avoid duplicating existing recipes
- Preserve all existing content

---

## Step 2: Stack Detection

Scan the project root for these signals. Check files in parallel where possible.

| Signal File | Stack | Implications |
|-------------|-------|-------------|
| `pyproject.toml` | Python | Check for `uv`, `poetry`, `hatch`, `setuptools` |
| `Cargo.toml` | Rust | `cargo build`, `cargo test`, `cargo clippy` |
| `package.json` | Node/JS/TS | Check for `pnpm-lock`, `yarn.lock`, `bun.lock` |
| `go.mod` | Go | `go build`, `go test`, `go vet` |
| `Gemfile` | Ruby | `bundle exec`, `rails`, `rake` |
| `composer.json` | PHP | `composer`, `php artisan` |
| `Dockerfile` / `docker-compose.yml` | Docker | Docker recipes |
| `alembic.ini` / `migrations/` | DB migrations | Database recipes |
| `.github/workflows/` | GitHub Actions | CI recipes |
| `Makefile` | Existing Make | Migration opportunity |
| `tauri.conf.json` | Tauri desktop | Desktop build recipes |
| `.pre-commit-config.yaml` | Pre-commit | Hook recipes |
| `mkdocs.yml` | MkDocs | Documentation recipes |
| `.env` / `.env.example` | Dotenv | `set dotenv-load` |

### Python Sub-detection

When `pyproject.toml` is found, determine the toolchain:

| Check | Tool | Recipe Style |
|-------|------|-------------|
| `uv.lock` or `[tool.uv]` | uv | `uv run --extra dev {cmd}` |
| `poetry.lock` or `[tool.poetry]` | poetry | `poetry run {cmd}` |
| `Pipfile` | pipenv | `pipenv run {cmd}` |
| None of above | pip | `python -m {cmd}` |

Check `pyproject.toml` for dev tools:
- `ruff` in deps → `fmt` and `lint` recipes
- `mypy` / `pyright` / `ty` in deps → `typecheck` recipe
- `pytest` in deps → `test` recipes

### Node Sub-detection

| Check | Manager | Recipe Style |
|-------|---------|-------------|
| `pnpm-lock.yaml` | pnpm | `pnpm {cmd}` |
| `yarn.lock` | yarn | `yarn {cmd}` |
| `bun.lock` / `bun.lockb` | bun | `bun {cmd}` |
| Only `package-lock.json` | npm | `npm run {cmd}` |

---

## Step 3: Recipe Selection

### Justfile Structure

Every justfile follows this order:

```just
# ── Settings ─────────────────────────────────────────────────────────

set dotenv-load                                    # if .env detected
set shell := ["bash", "-euo", "pipefail", "-c"]   # always

# ── Variables ────────────────────────────────────────────────────────

VERSION := `grep -m1 '^version' pyproject.toml | sed -E 's/version = "(.*)"/\1/'`

# ── Default ──────────────────────────────────────────────────────────

default:
    @just --list --unsorted

# ── {Category} ───────────────────────────────────────────────────────
# ... recipes grouped by category
```

### Section Divider Convention

Use unicode box-drawing characters, padded to column 72:

```
# ── Section Name ─────────────────────────────────────────────────────
```

### Recipe Categories (include only if stack-relevant)

Each category below includes the recipes to generate. Skip categories that don't apply.

#### Development (almost always included)

```just
# ── Development ──────────────────────────────────────────────────────

# Install project dependencies
install:
    {package-install-command}

# Run the development server
dev:
    {dev-server-command}

# Build for production
build:
    {build-command}

# Remove build artifacts
clean:
    -{clean-command}
```

#### Quality (when linter/formatter detected)

```just
# ── Quality ──────────────────────────────────────────────────────────

# Run all quality gates
qa: fmt-check lint typecheck test

# Format code
fmt:
    {format-command}

# Check formatting without changes
fmt-check:
    {format-check-command}

# Lint code
lint:
    {lint-command}

# Lint and auto-fix
lint-fix:
    {lint-fix-command}

# Type check
typecheck:
    {typecheck-command}
```

#### Testing (when test framework detected)

```just
# ── Testing ──────────────────────────────────────────────────────────

# Run tests
test *args:
    {test-command} {{args}}

# Run tests with coverage
test-cov:
    {test-coverage-command}

# Run a specific test file
test-file file *args:
    {test-command} {{file}} -v {{args}}

# Run tests matching a pattern
test-k pattern *args:
    {test-command} -k "{{pattern}}" -v {{args}}
```

#### Database (when migrations detected)

```just
# ── Database ─────────────────────────────────────────────────────────

[group('database')]
# Run database migrations
db-migrate:
    {migrate-command}

[group('database')]
# Rollback last migration
db-rollback:
    {rollback-command}

[group('database')]
[confirm("This will reset the database. Continue?")]
# Reset database (drop + create + migrate + seed)
db-reset:
    {reset-command}
```

#### Docker (when Dockerfile/compose detected)

```just
# ── Docker ───────────────────────────────────────────────────────────

[group('docker')]
# Start containers in background
up:
    docker compose up -d

[group('docker')]
# Stop containers
down:
    docker compose down

[group('docker')]
# Tail container logs
logs service="":
    docker compose logs -f {{service}}

[group('docker')]
# Rebuild and restart containers
rebuild: down
    docker compose build --no-cache
    docker compose up -d
```

#### Git & CI (always useful)

```just
# ── Git ──────────────────────────────────────────────────────────────

# Show repo status
status:
    @git status --short --branch
    @echo ""
    @git log --oneline -5
```

If GitHub Actions detected, add:

```just
# Watch CI checks on current PR
ci-watch:
    gh pr checks --watch

# Open current PR in browser
pr-open:
    gh pr view --web
```

#### Documentation (when docs tool detected)

```just
# ── Docs ─────────────────────────────────────────────────────────────

# Serve docs locally
docs-serve:
    {docs-serve-command}

# Build docs
docs-build:
    {docs-build-command}
```

#### Deploy (when deploy scripts/config detected)

```just
# ── Deploy ───────────────────────────────────────────────────────────

[confirm("Deploy to {{env}}?")]
# Deploy to target environment
deploy env="staging":
    ./scripts/deploy.sh {{env}}
```

---

## Recipe Writing Rules

Follow these rules for every recipe:

| Rule | Example |
|------|---------|
| Documentation comment above every public recipe | `# Run the test suite` |
| `@` prefix on informational/display recipes | `@just --list`, `@git status` |
| Use `*args` for passthrough flexibility | `test *args:` |
| Use parameter defaults for common variations | `serve port="8080":` |
| `[confirm]` on destructive operations | `[confirm("Drop database?")]` |
| `[group]` when 3+ recipes share a domain | `[group('docker')]` |
| `-` prefix to ignore errors on cleanup commands | `-rm -rf build/` |
| `[private]` or `_` prefix for helper recipes | `_ensure-deps:` |
| Use backticks for dynamic values | `` VERSION := `cmd` `` |
| `[no-exit-message]` on recipes where error output is noise | Clean/teardown recipes |

### Parameter Patterns

```just
# Required parameter
build target:
    cargo build --target {{target}}

# Optional with default
serve port="8080":
    python -m http.server {{port}}

# Variadic (zero or more)
test *args:
    pytest {{args}}

# Variadic (one or more)
run +files:
    cat {{files}}

# Exported as env var
deploy $ENV:
    ./deploy.sh
```

---

## Audit Checklist

When auditing an existing justfile, check each item:

| Check | Issue | Fix |
|-------|-------|-----|
| Has `default` recipe? | Users get error on bare `just` | Add `default: @just --list --unsorted` |
| Has `set shell`? | Inconsistent cross-platform behavior | Add `set shell := ["bash", "-euo", "pipefail", "-c"]` |
| Every public recipe documented? | `just --list` shows bare names | Add `#` comment above each |
| Destructive recipes have `[confirm]`? | Accidental data loss | Add `[confirm("...")]` |
| Uses `[group]` for 10+ recipes? | Wall of text in `--list` | Group by domain |
| Hardcoded paths? | Breaks across environments | Use variables or `env()` |
| Recipes over 15 lines? | Hard to maintain | Extract to script file |
| Duplicate logic across recipes? | Maintenance burden | Extract to `_helper` recipe |
| Uses `../` in paths? | Fragile on refactor | Use `justfile_directory()` |
| Has `set dotenv-load` when `.env` exists? | Manual env loading | Add `set dotenv-load` |
| Uses `[no-exit-message]` on cleanup? | Noisy error output | Add attribute |
| Variables at top of file? | Scattered declarations | Move to top after settings |

### Audit Output Format

```markdown
## Justfile Audit

**File:** `justfile` (42 recipes, 3 groups)

### Issues Found

| Severity | Issue | Line | Fix |
|----------|-------|------|-----|
| HIGH | No `[confirm]` on `db-reset` | 87 | Add `[confirm("Reset database?")]` |
| MEDIUM | Missing doc comment on `build` | 23 | Add `# Build the project` |
| LOW | Could group docker recipes | 45-72 | Add `[group('docker')]` |

### Recommendations
1. ...

### Score: 7/10
```

---

## Anti-Patterns

| Avoid | Why | Instead |
|-------|-----|---------|
| No `default` recipe | Confusing first experience | `default: @just --list --unsorted` |
| Missing `set shell` | Platform-dependent behavior | Always set explicitly |
| Giant recipes (15+ lines) | Unmaintainable | Extract to `scripts/` |
| Hardcoded secrets | Security risk | Use `env()` or dotenv |
| `../` in paths | Breaks on move/refactor | `justfile_directory()` |
| Undocumented recipes | Poor discoverability | Comment above every public recipe |
| No `[confirm]` on destructive ops | Accidental damage | Always confirm drops/resets/deploys |
| Duplicating Makefile patterns | `.PHONY`, tab-only, etc. | Use just idioms natively |
| Over-engineering | 100+ recipes nobody uses | Start with what you need |
| Mixing concerns in one recipe | Hard to compose | One purpose per recipe |

---

## Verification

After generating a justfile:

- [ ] `just` (bare) shows organized recipe list
- [ ] Every public recipe has a doc comment
- [ ] `set shell` is present
- [ ] Destructive recipes have `[confirm]`
- [ ] Variables use backticks for dynamic values
- [ ] No hardcoded paths or secrets
- [ ] Recipes match detected stack tools
- [ ] Section dividers are consistent
- [ ] `just --fmt --check` passes (if just is installed)

---

<details>
<summary><strong>Deep Dive: Stack-Specific Templates</strong></summary>

### Python + uv

```just
set dotenv-load
set shell := ["bash", "-euo", "pipefail", "-c"]

VERSION := `grep -m1 '^version' pyproject.toml | sed -E 's/version = "(.*)"/\1/'`

default:
    @just --list --unsorted

# ── Development ──────────────────────────────────────────────────────

# Install all dependencies
install:
    uv sync --all-extras

# Build package
build:
    uv build

# ── Quality ──────────────────────────────────────────────────────────

# Run all quality gates
qa: fmt-check lint typecheck test

# Format code
fmt:
    uv run --extra dev ruff format .

# Check formatting
fmt-check:
    uv run --extra dev ruff format --check .

# Lint code
lint:
    uv run --extra dev ruff check .

# Lint and auto-fix
lint-fix:
    uv run --extra dev ruff check --fix .

# Type check
typecheck:
    uv run --extra dev mypy .

# ── Testing ──────────────────────────────────────────────────────────

# Run tests
test *args:
    uv run --extra dev pytest {{args}}

# Run tests with coverage
test-cov:
    uv run --extra dev pytest --cov=src --cov-report=term-missing

# Run a specific test file
test-file file *args:
    uv run --extra dev pytest {{file}} -v {{args}}

# Run tests matching a pattern
test-k pattern *args:
    uv run --extra dev pytest -k "{{pattern}}" -v {{args}}

# ── Git ──────────────────────────────────────────────────────────────

# Show repo status
status:
    @git status --short --branch
    @echo ""
    @git log --oneline -5
```

### Rust + Cargo

```just
set shell := ["bash", "-euo", "pipefail", "-c"]

VERSION := `grep -m1 '^version' Cargo.toml | sed -E 's/version = "(.*)"/\1/'`

default:
    @just --list --unsorted

# ── Development ──────────────────────────────────────────────────────

# Build in release mode
build:
    cargo build --release

# Run the project
run *args:
    cargo run -- {{args}}

# Remove build artifacts
clean:
    cargo clean

# ── Quality ──────────────────────────────────────────────────────────

# Run all checks
qa: fmt-check lint test

# Format code
fmt:
    cargo fmt

# Check formatting
fmt-check:
    cargo fmt --check

# Lint with clippy
lint:
    cargo clippy -- -D warnings

# ── Testing ──────────────────────────────────────────────────────────

# Run tests
test *args:
    cargo test {{args}}

# Run tests with output
test-verbose:
    cargo test -- --nocapture

# Run a specific test
test-name name:
    cargo test {{name}} -- --nocapture

# ── Git ──────────────────────────────────────────────────────────────

# Show repo status
status:
    @git status --short --branch
    @echo ""
    @git log --oneline -5
```

### Node/TypeScript + pnpm

```just
set dotenv-load
set shell := ["bash", "-euo", "pipefail", "-c"]

default:
    @just --list --unsorted

# ── Development ──────────────────────────────────────────────────────

# Install dependencies
install:
    pnpm install

# Run dev server
dev:
    pnpm dev

# Build for production
build:
    pnpm build

# Remove build artifacts
clean:
    -rm -rf dist/ node_modules/.cache/

# ── Quality ──────────────────────────────────────────────────────────

# Run all checks
qa: lint typecheck test

# Lint code
lint:
    pnpm lint

# Lint and auto-fix
lint-fix:
    pnpm lint --fix

# Type check
typecheck:
    pnpm tsc --noEmit

# Format code
fmt:
    pnpm prettier --write .

# Check formatting
fmt-check:
    pnpm prettier --check .

# ── Testing ──────────────────────────────────────────────────────────

# Run tests
test *args:
    pnpm test {{args}}

# Run tests in watch mode
test-watch:
    pnpm test --watch

# Run tests with coverage
test-cov:
    pnpm test --coverage

# ── Git ──────────────────────────────────────────────────────────────

# Show repo status
status:
    @git status --short --branch
    @echo ""
    @git log --oneline -5
```

### Go

```just
set shell := ["bash", "-euo", "pipefail", "-c"]

MODULE := `head -1 go.mod | awk '{print $2}'`

default:
    @just --list --unsorted

# ── Development ──────────────────────────────────────────────────────

# Build the binary
build:
    go build -o bin/ ./...

# Run the project
run *args:
    go run . {{args}}

# Remove build artifacts
clean:
    -rm -rf bin/

# ── Quality ──────────────────────────────────────────────────────────

# Run all checks
qa: vet lint test

# Vet code
vet:
    go vet ./...

# Lint with golangci-lint
lint:
    golangci-lint run

# Format code
fmt:
    gofmt -w .

# ── Testing ──────────────────────────────────────────────────────────

# Run tests
test *args:
    go test ./... {{args}}

# Run tests with coverage
test-cov:
    go test -coverprofile=coverage.out ./...
    go tool cover -html=coverage.out -o coverage.html

# Run tests with race detection
test-race:
    go test -race ./...

# Generate mocks
mock:
    go generate ./...

# ── Git ──────────────────────────────────────────────────────────────

# Show repo status
status:
    @git status --short --branch
    @echo ""
    @git log --oneline -5
```

### Ruby + Rails

```just
set dotenv-load
set shell := ["bash", "-euo", "pipefail", "-c"]

default:
    @just --list --unsorted

# ── Development ──────────────────────────────────────────────────────

# Install dependencies
install:
    bundle install

# Start Rails server
dev:
    bin/rails server

# Open Rails console
console:
    bin/rails console

# ── Quality ──────────────────────────────────────────────────────────

# Run all checks
qa: lint test

# Lint with RuboCop
lint:
    bundle exec rubocop

# Lint and auto-fix
lint-fix:
    bundle exec rubocop -A

# ── Testing ──────────────────────────────────────────────────────────

# Run tests
test *args:
    bundle exec rspec {{args}}

# Run a specific test file
test-file file *args:
    bundle exec rspec {{file}} {{args}}

# ── Database ─────────────────────────────────────────────────────────

[group('database')]
# Run migrations
db-migrate:
    bin/rails db:migrate

[group('database')]
# Rollback last migration
db-rollback:
    bin/rails db:rollback

[group('database')]
[confirm("Reset the database?")]
# Drop, create, migrate, and seed
db-reset:
    bin/rails db:reset

[group('database')]
# Seed the database
db-seed:
    bin/rails db:seed

# ── Git ──────────────────────────────────────────────────────────────

# Show repo status
status:
    @git status --short --branch
    @echo ""
    @git log --oneline -5
```

</details>

<details>
<summary><strong>Deep Dive: Advanced Patterns</strong></summary>

### OS-Specific Recipes

```just
[linux]
install-deps:
    sudo apt install -y libssl-dev pkg-config

[macos]
install-deps:
    brew install openssl pkg-config
```

### Module Organization (large projects)

```just
# justfile (root)
mod docker 'ops/docker.just'
mod db 'ops/database.just'
mod deploy 'ops/deploy.just'

default:
    @just --list --unsorted
```

### CI/CD Integration

```just
# Recipe that mirrors CI pipeline locally
ci: fmt-check lint typecheck test build
    @echo "All CI checks passed locally"

# Watch CI status
ci-watch:
    gh pr checks --watch

# Re-run failed CI job
ci-retry:
    gh run rerun --failed
```

### Multi-Service Projects

```just
# Run all services (needs [script] so &/wait share one shell)
[script("bash")]
dev-all:
    set -euo pipefail
    just frontend dev &
    just backend dev &
    wait

[working-directory("frontend")]
frontend *args:
    pnpm {{args}}

[working-directory("backend")]
backend *args:
    uv run {{args}}
```

### Release Workflows

```just
VERSION := `grep -m1 '^version' pyproject.toml | sed -E 's/version = "(.*)"/\1/'`

# Tag and push a release
[confirm("Release v{{VERSION}}?")]
release: qa
    git tag -a "v{{VERSION}}" -m "Release v{{VERSION}}"
    git push origin "v{{VERSION}}"

# Bump version (requires bump2version or similar)
bump level="patch":
    bump2version {{level}}
```

### Colorized Output

```just
# Use built-in ANSI constants
_success msg:
    @echo "{{GREEN}}{{BOLD}}OK{{NORMAL}} {{msg}}"

_error msg:
    @echo "{{RED}}{{BOLD}}ERROR{{NORMAL}} {{msg}}"
```

### Help Recipe Alternative

```just
# If you want more control than --list:
[private]
default:
    @echo "{{BOLD}}Available commands:{{NORMAL}}"
    @echo ""
    @just --list --unsorted --list-heading ''
    @echo ""
    @echo "Run {{CYAN}}just <recipe> --help{{NORMAL}} for details"
```

</details>

<details>
<summary><strong>Deep Dive: Justfile Syntax Reference</strong></summary>

### Settings Reference

| Setting | Purpose | Default |
|---------|---------|---------|
| `set shell := [...]` | Shell for recipe lines | `["sh", "-cu"]` |
| `set dotenv-load` | Load `.env` file | `false` |
| `set dotenv-filename := "..."` | Custom dotenv filename | `.env` |
| `set dotenv-required` | Error if `.env` missing | `false` |
| `set export` | Export all variables as env | `false` |
| `set positional-arguments` | Enable `$1`, `$@` | `false` |
| `set fallback` | Search parent dirs | `false` |
| `set ignore-comments` | Don't pass `#` to shell | `false` |
| `set quiet` | Suppress command echo globally | `false` |
| `set working-directory := "..."` | Global working dir | justfile dir |
| `set tempdir := "..."` | Temp file location | system default |
| `set script-interpreter := [...]` | Default for `[script]` | none |
| `set unstable` | Enable unstable features | `false` |

### Attribute Reference

| Attribute | Scope | Purpose |
|-----------|-------|---------|
| `[confirm]` / `[confirm("msg")]` | Recipe | Require Y/N before running |
| `[doc("text")]` | Recipe/Module | Set documentation string |
| `[group("name")]` | Recipe/Module | Assign to group in `--list` |
| `[linux]` `[macos]` `[windows]` `[unix]` | Recipe | OS-specific |
| `[no-cd]` | Recipe | Don't cd to justfile dir |
| `[no-exit-message]` | Recipe | Suppress failure output |
| `[no-quiet]` | Recipe | Override global quiet |
| `[private]` | Recipe | Hide from `--list` |
| `[script]` / `[script("interp")]` | Recipe | Run as temp script file |
| `[extension("ext")]` | Recipe | Temp file extension |
| `[positional-arguments]` | Recipe | Enable `$1`, `$@` |
| `[parallel]` | Recipe | Run dependencies concurrently |
| `[working-directory("path")]` | Recipe | Override working dir |

### Built-in Functions (Most Useful)

**System:** `arch()`, `os()`, `os_family()`, `num_cpus()`

**Paths:** `justfile()`, `justfile_directory()`, `invocation_directory()`, `absolute_path(p)`, `extension(p)`, `file_name(p)`, `file_stem(p)`, `parent_directory(p)`, `join(a, b)`

**Filesystem:** `path_exists(p)`, `read(p)`

**Environment:** `env("KEY")`, `env("KEY", "default")`, `shell("cmd")`

**String:** `quote(s)`, `replace(s, from, to)`, `replace_regex(s, re, to)`, `trim(s)`, `uppercase(s)`, `lowercase(s)`, `snakecase(s)`, `kebabcase(s)`

**Utility:** `uuid()`, `sha256(s)`, `datetime(fmt)`, `datetime_utc(fmt)`, `semver_matches(ver, req)`, `error("msg")`

**Constants:** `HEX`, `HEXLOWER`, `RED`, `GREEN`, `BLUE`, `BOLD`, `NORMAL` (ANSI colors)

### CLI Reference

```bash
just                     # Run default recipe
just recipe              # Run named recipe
just recipe arg1 arg2    # With arguments
just --list              # Show available recipes (grouped)
just --list --unsorted   # Show in file order
just --summary           # Compact one-line list
just --show recipe       # Show recipe source
just --dry-run recipe    # Show without executing
just --fmt               # Auto-format justfile
just --fmt --check       # Check formatting (CI, exit 1 if unformatted)
just --check             # Check justfile syntax
just --choose            # Interactive picker (fzf)
just --dump              # Dump resolved justfile
just --dump --format json # Machine-readable dump
just --evaluate          # Print all variables
just --evaluate var      # Print one variable
just --completions zsh   # Generate shell completions
just --yes recipe        # Skip [confirm] prompts
just mod::recipe         # Run module recipe
```

</details>

---

## Extension Points

1. **New stack templates** — Add to the Stack-Specific Templates section
2. **Custom recipe categories** — Add alongside existing categories
3. **Organization-specific conventions** — Override divider style, naming patterns
4. **Module patterns** — Templates for multi-module justfile organization
