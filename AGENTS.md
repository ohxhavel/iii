# AGENTS.md

You are working in the iii monorepo — a backend unification engine with three primitives: **Function**, **Trigger**, **Worker**. The engine is Rust. SDKs exist for TypeScript, Python, Rust, and Go. All communicate over WebSocket.

For the long form of everything below — engine layout, testing setup, CI gates, docs versioning, release flow — see [CLAUDE.md](CLAUDE.md).

## Commands

The `Makefile` mirrors the CI jobs and is the shortest path to a green build:

```bash
make install                     # pnpm install + uv sync (Python SDK)
make install-hooks               # pre-commit cargo fmt check
make fix                         # auto-format and auto-fix lint, all languages
make check                       # lint + fmt + typecheck + build
make ci-local                    # everything CI runs
make cli-docs                    # regenerate docs/next/cli-reference/ (CI fails if stale)
make engine-up / engine-down     # test engine for SDK integration tests
```

The underlying tools:

```bash
# Setup
pnpm install                     # JS/TS dependencies
cargo build --release            # Rust workspace

# Build
pnpm build                       # all JS/TS packages (Turborepo)
cargo build --release             # engine + Rust SDK + console

# Test
pnpm test                        # all JS/TS tests
cargo test                       # all Rust tests
cargo test -p iii                 # engine only
cargo test -p iii-sdk             # Rust SDK only
cd sdk/packages/python/iii && uv sync --extra dev && uv run pytest  # Python SDK
cd sdk/packages/go/iii && go test ./...                            # Go SDK
make engine-test                 # engine suite (installs iii-worker to PATH first)

# Lint & Format
pnpm fmt                         # format JS/TS (Biome)
pnpm fmt:check                   # check without changes
pnpm lint                        # lint JS/TS
cargo fmt --all                   # format Rust
cargo clippy --workspace          # lint Rust

# Run
cargo run --release               # start engine (reads engine/config.yaml)
pnpm dev:console                  # console frontend dev server
pnpm dev:docs                     # docs dev server (Mintlify)
pnpm dev:website                  # website dev server

# Cloud — passthrough to the external iii-cloud binary (iii-hq/iii-cloud-cli)
iii cloud deploy --config <path>  # deploy to iii Cloud
iii cloud list                    # list deployments
iii cloud update <deployment-id>  # update a deployment
iii cloud delete <deployment-id>  # delete a deployment
```

## Project Map

```
engine/                          Rust engine — runtime, built-in workers, protocol, CLI
crates/                          Supporting Rust crates — sandbox/VM runtime, tooling, scaffolding
sdk/packages/node/iii/           TypeScript SDK (npm: iii-sdk)
sdk/packages/node/iii-browser/   Browser SDK (npm: iii-browser-sdk)
sdk/packages/python/iii/         Python SDK (PyPI: iii-sdk)
sdk/packages/rust/iii/           Rust SDK (crates.io: iii-sdk)
sdk/packages/go/iii/             Go SDK (github.com/iii-hq/iii/sdk/packages/go/iii)
console/                         Developer console (React + Rust)
skills/                          Agent skills (auto-discovered by SkillKit)
docs/                            Documentation site (Mintlify/MDX) — author under docs/next/
website/                         iii.dev website
website/roadmap/                 Tech-spec decks and site tooling (iii.dev/roadmap/) — SOP in its README
tech-specs/                      Markdown-only specs; frontmatter in each README.md lists it on the site
scripts/                         Build and CI scripts
infra/                           Terraform for the website/CDN
```

**Workspaces:** `Cargo.toml` (Rust), `pnpm-workspace.yaml` (JS/TS), `turbo.json` (build orchestration). Every manifest is versioned in lockstep — never bump one on its own.

## Boundaries

### Always

- Use `pnpm` (never `npm`) for JS/TS packages
- Use `cargo fmt --all` before committing Rust changes
- Use `pnpm fmt` before committing JS/TS changes
- Use leading slashes for HTTP `api_path` values: `/orders`, `/users/:id`
- Use `expression` (not `cron`) for cron trigger config fields
- Use `::` separator for function IDs: `orders::validate`, `reports::daily-summary`
- Use `workspace:*` for internal pnpm package references
- Include `## When to Use` and `## Boundaries` sections in every SKILL.md
- Match SKILL.md `name` field to its directory name exactly

### Ask First

- Changes to public SDK APIs (npm/PyPI/crates.io/Go module surface)
- Changes to engine config schema (`engine/config.yaml`)
- Changes to CI/CD workflows (`.github/`)
- Adding new engine modules
- Modifying the WebSocket protocol between SDK and engine

### Never

- Commit secrets, API keys, or credentials
- Use `npm` instead of `pnpm`
- Push directly to `main`
- Change engine licensing (ELv2) or SDK licensing (Apache-2.0)
- Remove "When to Use" / "Boundaries" from SKILL.md files (SkillKit validates these)
- Use `cron` as a config key — the engine uses `expression`
- Omit leading slashes on `api_path` — the engine standard is `/path`

## Code Style

**Rust (engine + SDK):**
```rust
// Function IDs use :: separator
iii.register_function(
    RegisterFunction::new("orders::validate", validate_order)
        .description("Validate an incoming order"),
);

// HTTP triggers use leading slash
iii.register_trigger(
    IIITrigger::Http(HttpTriggerConfig::new("/orders/validate").method(HttpMethod::Post))
        .for_function("orders::validate"),
);

// Cron triggers use `expression` field (7-field: sec min hour dom month dow year)
iii.register_trigger(
    IIITrigger::Cron(CronTriggerConfig::new("0 0 9 * * * *"))
        .for_function("reports::daily-summary"),
);
```

**TypeScript (SDK):**
```typescript
// HTTP trigger with leading slash
iii.registerTrigger({
  type: 'http',
  function_id: 'orders::validate',
  config: { api_path: '/orders/validate', http_method: 'POST' },
});

// HTTP trigger with middleware chain
iii.registerTrigger({
  type: 'http',
  function_id: 'orders::validate',
  config: {
    api_path: '/orders/validate',
    http_method: 'POST',
    middleware_function_ids: ['middleware::auth', 'middleware::rate-limit'],
  },
});

// Cron trigger with `expression` (not `cron`)
iii.registerTrigger({
  type: 'cron',
  function_id: 'reports::daily-summary',
  config: { expression: '0 0 9 * * * *' },
});

// Trigger with metadata (optional, stored with the trigger)
iii.registerTrigger({
  type: 'cron',
  function_id: 'reports::daily-summary',
  config: { expression: '0 0 9 * * * *' },
  metadata: { owner: 'billing-team', priority: 'high' },
});
```

**Python (SDK):**
```python
# Same patterns — leading slash, expression field
iii.register_trigger({
    "type": "http",
    "function_id": "orders::validate",
    "config": {"api_path": "/orders/validate", "http_method": "POST"},
})
```

## Skills

The `skills/` directory contains six knowledge skills — `iii-getting-started`, `iii-core-primitives`, `iii-sdk-reference`, `iii-engine-config`, `iii-architecture-patterns`, `iii-error-handling` — plus the `presentation` tool skill. Install with `npx skills add iii-hq/iii/skills` (add `--skill <name>` for one).

Worker-specific capability skills deliberately live with their workers in [iii-hq/workers](https://github.com/iii-hq/workers), not here. Do not duplicate queue, pub/sub, state, cron, stream, or observability material into `skills/`.

## Blog (agent knowledge base)

Published at https://iii.dev/blog/ — architecture posts and worked examples for coding agents. Each post is available as markdown:

- Index: https://iii.dev/blog/index.md
- Posts: https://iii.dev/blog/<slug>.md (every slug listed in the index)

Source lives in `website/src/content/blog/`. Also indexed in https://iii.dev/llms.txt and https://iii.dev/AGENTS.md.

## Licensing

- `engine/` — Elastic License v2 (ELv2)
- Everything else — Apache-2.0
