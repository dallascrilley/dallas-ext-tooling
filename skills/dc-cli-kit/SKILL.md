---
name: dc-cli-kit
description: Build TypeScript CLIs with dc-cli-kit wrapping incur — covers createCli, config, auth, HTTP, errors, middleware, and full command definition patterns
version: 0.1.0
---

# Building CLIs with dc-cli-kit + incur

dc-cli-kit is an opinionated wrapper around the [incur](https://github.com/wevm/incur) CLI framework. It standardizes auth, config, HTTP, and error patterns so each CLI is pure domain logic. Always import from `dc-cli-kit` — it re-exports incur's full API.

## Architecture

```
dc-cli-kit (you import this)
  └── incur (re-exported)
       ├── Cli.create() → .command() → .serve()
       ├── z (zod)
       ├── middleware()
       ├── Errors
       └── Formatter, Mcp, Schema
```

**Your CLI code** only touches domain logic. dc-cli-kit handles:
- Config resolution (env → .env → project → XDG → legacy → defaults)
- Auth middleware (API key, OAuth, 1Password)
- Retryable HTTP with error mapping
- Standardized errors + exit codes
- Timing/debug middleware

## Quick Start Template

```ts
import { createCli, z, apiKeyAuth, createClient } from 'dc-cli-kit'

const cli = createCli('mycli', {
  description: 'My CLI tool',
  version: '0.1.0',
  vars: z.object({
    auth: z.string().optional(),
    config: z.object({
      baseUrl: z.string(),
    }).optional(),
  }),
  config: {
    schema: z.object({
      baseUrl: z.string().default('https://api.example.com/v1'),
    }),
  },
  auth: apiKeyAuth({ envVar: 'MYCLI_API_KEY' }),
})

cli.command('list', {
  description: 'List resources',
  options: z.object({
    limit: z.coerce.number().default(25).describe('Max results'),
    format: z.enum(['table', 'ids']).default('table').describe('Output style'),
  }),
  alias: { limit: 'l' },
  output: z.object({
    items: z.array(z.object({ id: z.string(), name: z.string() })),
    total: z.number(),
  }),
  examples: [
    { description: 'List first 10' , options: { limit: 10 } },
  ],
  run(c) {
    const client = createClient({
      baseUrl: c.var.config.baseUrl,
      authProvider: () => c.var.auth,
    })
    return client.get('/resources', {
      headers: { 'X-Page-Size': String(c.options.limit) },
    }).then(r => r.data)
  },
})

cli.serve()
```

## Core Patterns

### Single-Command CLI

Pass `run` directly to `createCli` options (via incur's `Cli.create`):

```ts
import { Cli, z } from 'dc-cli-kit'

Cli.create('greet', {
  description: 'Greet someone',
  args: z.object({ name: z.string().describe('Name to greet') }),
  run(c) {
    return { message: `hello ${c.args.name}` }
  },
}).serve()
```

### Multi-Command CLI (Router)

Use `createCli` (no `run`), then chain `.command()`:

```ts
const cli = createCli('deploy', {
  description: 'Deployment CLI',
  vars: z.object({ auth: z.string().optional() }),
  auth: apiKeyAuth({ envVar: 'DEPLOY_TOKEN' }),
})

cli.command('status', {
  description: 'Check deployment status',
  args: z.object({ id: z.string().describe('Deploy ID') }),
  run(c) {
    return { id: c.args.id, status: 'running' }
  },
})

cli.command('rollback', {
  description: 'Rollback a deployment',
  args: z.object({ id: z.string().describe('Deploy ID') }),
  options: z.object({ force: z.boolean().optional().describe('Skip confirmation') }),
  alias: { force: 'f' },
  run(c) {
    return { rolledBack: c.args.id }
  },
})

cli.serve()
```

### Subcommand Groups (Nested CLIs)

Mount a separate `Cli` instance as a command group:

```ts
import { Cli, z } from 'dc-cli-kit'

const pr = Cli.create('pr', { description: 'Pull request commands' })
  .command('list', {
    description: 'List PRs',
    options: z.object({
      state: z.enum(['open', 'closed', 'all']).default('open'),
    }),
    run(c) {
      return { prs: [], state: c.options.state }
    },
  })
  .command('create', {
    description: 'Create a PR',
    args: z.object({ title: z.string().describe('PR title') }),
    run(c) {
      return { url: 'https://github.com/repo/pull/1' }
    },
  })

const cli = createCli('gh', { description: 'GitHub CLI' })
cli.command(pr) // mounts as `gh pr list`, `gh pr create`
cli.serve()
```

## Command Definition Reference

Every field available on `.command()`:

```ts
cli.command('deploy', {
  // --- Schemas (all optional) ---
  args: z.object({ env: z.enum(['staging', 'prod']).describe('Target') }),
  options: z.object({
    force: z.boolean().optional().describe('Skip confirmation'),
    tag: z.string().default('latest').describe('Tag'),
  }),
  env: z.object({
    DEPLOY_TOKEN: z.string().describe('Auth token'),
  }),
  output: z.object({ id: z.string(), url: z.string() }),

  // --- Metadata ---
  description: 'Deploy the app',
  alias: { force: 'f', tag: 't' },
  hint: 'Requires DEPLOY_TOKEN',
  format: 'toon', // default output format
  outputPolicy: 'all', // 'all' | 'agent-only'

  // --- Documentation ---
  examples: [
    { args: { env: 'staging' }, description: 'Deploy to staging' },
    { args: { env: 'prod' }, options: { force: true }, description: 'Force to prod' },
  ],

  // --- Per-command middleware ---
  middleware: [requireAdmin],

  // --- Handler ---
  run(c) {
    // c.args.env      — typed from schema
    // c.options.force  — typed from schema
    // c.env.DEPLOY_TOKEN — typed from schema
    // c.var.auth       — typed from CLI vars
    // c.agent          — true if piped / non-TTY
    // c.ok(data, { cta }) — explicit success with CTAs
    // c.error({ code, message, retryable, cta }) — explicit error
    return { id: 'd-123', url: 'https://...' }
  },
})
```

## Config (`loadConfig`)

6-layer resolution chain. Higher layers override lower:

| Priority | Source | Path |
|----------|--------|------|
| 1 | Env vars | `MYCLI_BASE_URL=...` |
| 2 | `.env` file | `.env` in cwd |
| 3 | Project config | `./mycli.config.json` |
| 4 | XDG config | `~/.config/mycli/config.json` |
| 5 | Legacy dotfile | `~/.myclirc` (opt-in) |
| 6 | Schema defaults | `z.string().default(...)` |

```ts
import { loadConfig, z } from 'dc-cli-kit'

const config = loadConfig('mycli', z.object({
  baseUrl: z.string().default('https://api.example.com'),
  timeout: z.coerce.number().default(30),
}), {
  configFormat: 'yaml',              // 'json' | 'yaml' | 'toml'
  envPrefix: 'MC_',                  // default: MYCLI_
  legacyPaths: ['~/.myclirc'],       // optional legacy support
})
```

Auto-loaded by `createCli` when `config.schema` is provided — sets `context.var.config`.

## Auth Middleware

All auth factories are **lazy** — credentials resolve inside `run()`, not at CLI init. This avoids latency on `--help`.

### API Key

```ts
import { apiKeyAuth } from 'dc-cli-kit'

apiKeyAuth({
  envVar: 'MY_API_KEY',            // required
  header: 'X-API-Key',            // default: "Authorization"
})
// Sets context.var.auth = "the-key-string"
```

### OAuth

```ts
import { oauthAuth } from 'dc-cli-kit'

oauthAuth({
  tokenPath: '~/.config/mycli/tokens.json',  // required
  refreshUrl: 'https://oauth.example.com/token',
  clientIdEnv: 'MYCLI_CLIENT_ID',
  clientSecretEnv: 'MYCLI_CLIENT_SECRET',
  expiryBufferMs: 5 * 60 * 1000,          // default: 5 min
})
// Sets context.var.auth = "access-token-string"
// Auto-refreshes expired tokens and persists to disk (0o600)
```

Token file format: `{ "accessToken": "...", "refreshToken": "...", "expiresAt": 1709337600000 }`

### 1Password

```ts
import { opsAuth } from 'dc-cli-kit'

opsAuth({
  vault: 'Engineering',   // required
  item: 'My API',         // required
  field: 'api_key',       // default: "password"
})
// Reads op://Engineering/My API/api_key via execFileSync (no shell)
// Sets context.var.auth = "secret-value"
```

## HTTP Client (`createClient`)

Retryable client on native `fetch()`. Zero extra deps.

```ts
import { createClient } from 'dc-cli-kit'

const client = createClient({
  baseUrl: 'https://api.example.com/v1',
  authProvider: () => context.var.auth,  // called per-request
  timeout: 30_000,
  retry: {
    maxRetries: 3,
    baseDelayMs: 1000,
    multiplier: 2,
    retryableStatuses: [429, 502, 503, 504],
  },
})

// Methods: get, post, put, patch, delete, request
const res = await client.get<User[]>('/users')
// res = { ok: true, status: 200, headers: Headers, data: User[] }

await client.post('/users', { body: { name: 'Alice' } })
await client.put('/users/1', { body: { name: 'Bob' } })
await client.delete('/users/1')
```

**Retry rules:**
- GET/PUT/DELETE/HEAD/OPTIONS: auto-retry on 5xx
- POST/PATCH: **never** auto-retry on 5xx (safe default)
- 429: **always** retried (request wasn't processed), respects `Retry-After`
- Opt in: `{ retryNonIdempotent: true }` per-request

**Status code to error mapping:**
- 401/403 → `AuthError`
- 404 → `NotFoundError`
- 429 → `RateLimitError` (reads `Retry-After`)
- 5xx → `IncurError` with `retryable: true`

## Error Types

All extend `Errors.IncurError` (from incur). Each has `code`, `message`, `hint`, `retryable`, `walk()`.

```ts
import { ConfigError, AuthError, NotFoundError, RateLimitError, ExitCode } from 'dc-cli-kit'

throw new AuthError({
  message: 'Token expired',
  hint: 'Run `mycli login`',
  code: 'TOKEN_EXPIRED',  // default: AUTH_ERROR
})

// Exit codes
ExitCode.Success      // 0
ExitCode.GeneralError // 1
ExitCode.UsageError   // 2
ExitCode.AuthError    // 3
ExitCode.NotFound     // 4
ExitCode.RateLimited  // 5
```

## Middleware

```ts
import { timing, debug, redactHeaders, sanitizeUrl } from 'dc-cli-kit'
```

- `timing` — logs `[timing] 142ms` to stderr after every command
- `debug` — logs `[debug] command: people list` when `--verbose` or `DEBUG=1`
- `redactHeaders({...})` — replaces `Authorization`, `X-API-Key`, `Cookie`, etc. with `***REDACTED***`
- `sanitizeUrl(url)` — masks `?token=...`, `?api_key=...`, etc.

Both `timing` and `debug` are auto-applied by `createCli`. Disable with `disableTiming: true` / `disableDebug: true`.

Custom middleware:

```ts
import { middleware, z } from 'dc-cli-kit'

const requireRole = middleware<typeof cli.vars>(async (c, next) => {
  if (!c.var.auth) throw new AuthError({ message: 'Not authenticated' })
  await next()
})

cli.use(requireRole)
// or per-command:
cli.command('admin', { middleware: [requireRole], run(c) { ... } })
```

## incur Features (Available via dc-cli-kit)

### Call-to-Actions (CTAs)

Guide agents and humans to next steps:

```ts
run(c) {
  return c.ok({ deployed: true }, {
    cta: {
      description: 'Next steps:',
      commands: [
        { command: 'status d-123', description: 'Check status' },
        'rollback d-123',
      ],
    },
  })
}
```

### Streaming (Async Generators)

```ts
cli.command('sync', {
  async *run() {
    yield { progress: 25 }
    yield { progress: 50 }
    yield { progress: 100, done: true }
  },
})
```

### Output Formats

All CLIs get `--format`, `--json`, `--verbose` for free:

```sh
mycli list                    # TOON (default, token-efficient)
mycli list --json             # JSON
mycli list --format yaml      # YAML
mycli list --format md        # Markdown tables
mycli list --verbose          # Full output envelope with metadata
```

### Agent Discovery

```sh
mycli skills add   # Sync skill files to Claude Code / Cursor
mycli mcp add      # Register as MCP server
mycli --llms       # Print machine-readable command manifest
mycli --mcp        # Start as MCP stdio server
```

### Testing CLIs

```ts
import { describe, expect, it } from 'vitest'

it('lists resources', async () => {
  let output = ''
  await cli.serve(['list', '--json'], {
    stdout: (s) => { output += s },
    exit: () => {},
    env: { MYCLI_API_KEY: 'test-key' },
  })
  expect(JSON.parse(output).data.items).toHaveLength(3)
})
```

## Project Structure Convention

```
my-cli/
├── package.json            # "dc-cli-kit" as dependency
├── tsconfig.json           # ESM, strict, NodeNext
├── vitest.config.ts
├── src/
│   ├── cli.ts              # createCli + commands
│   └── index.ts            # export cli; cli.serve()
└── test/
    └── cli.test.ts
```

Minimal `package.json`:

```json
{
  "name": "my-cli",
  "type": "module",
  "bin": { "my-cli": "./dist/index.js" },
  "dependencies": { "dc-cli-kit": "^0.1.0" },
  "peerDependencies": { "zod": "^4.3.6" },
  "devDependencies": {
    "typescript": "latest",
    "vitest": "latest",
    "zile": "latest",
    "zod": "^4.3.6"
  },
  "scripts": {
    "build": "zile",
    "test": "vitest"
  }
}
```
