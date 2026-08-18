# CLAUDE.md

Guidance for Claude Code and other coding agents working in the **iii** monorepo.

`AGENTS.md` is the short version of this file and stays in sync with it. `CONTRIBUTING.md` covers
licensing of contributions, `.github/workflows/WORKFLOWS.md` covers CI/CD in depth, and
`docs/RELEASING.md` covers docs versioning.

## What this repo is

iii is a backend unification engine built on three primitives:

- **Worker** — a process that connects to the engine and registers functions and triggers. Any
  service in any supported language is a worker. Workers can create other workers at runtime.
- **Trigger** — anything that causes a function to run: HTTP, cron, queue, state change, stream
  event, direct call, or a custom trigger type.
- **Function** — a unit of work with a stable `namespace::name` identifier, living in a worker.

The engine is Rust. SDKs exist for **TypeScript/Node, Python, Rust, and Go**, and all of them speak
the same WebSocket protocol to the engine (`engine/src/protocol.rs`).

Everything in the workspace is versioned in lockstep from `Cargo.toml`'s
`[workspace.package] version` — currently `0.23.0-rc.1` — mirrored into every `package.json`,
`pyproject.toml` (PEP 440 form, e.g. `0.23.0rc1`), and the Go SDK's `sdkVersion` constant. Never
bump a single manifest by hand; `.github/scripts/bump_manifests.py` does it for all of them.

## Repository map

| Path                   | What it is                                                                    |
| ---------------------- | ----------------------------------------------------------------------------- |
| `engine/`              | The iii engine: runtime, built-in workers, protocol, CLI (`engine/README.md`) |
| `engine/src/workers/`  | Built-in workers, each with its own README                                    |
| `crates/`              | Supporting Rust crates — sandbox/VM runtime, tooling, scaffolding             |
| `sdk/packages/node/`   | `iii-sdk` (npm), `iii-browser-sdk`, `@iii-dev/helpers`, observability         |
| `sdk/packages/python/` | `iii-sdk` (PyPI), helpers, observability                                      |
| `sdk/packages/rust/`   | `iii-sdk` (crates.io), helpers, observability                                 |
| `sdk/packages/go/`     | Go SDK (`github.com/iii-hq/iii/sdk/packages/go/iii`)                          |
| `sdk/fixtures/`        | Engine configs used by SDK integration tests                                  |
| `console/`             | Developer console — `console-frontend` (React) + `console-rust` (binary)      |
| `skills/`              | Agent skills published via `npx skills add iii-hq/iii/skills`                 |
| `docs/`                | Mintlify docs site — versioned (see **Docs**)                                 |
| `website/`             | iii.dev website (Astro); `website/roadmap/` builds `iii.dev/roadmap/`         |
| `tech-specs/`          | Markdown-only specs, one folder per spec, listed via README frontmatter       |
| `scripts/`             | `generate-cli-docs.sh`, `generate-api-docs.sh`, `start-iii.sh`, `stop-iii.sh` |
| `infra/terraform/`     | Website/CDN infrastructure (Terraform)                                        |
| `.github/`             | Workflows and release scripts                                                 |

### Crates

`iii-worker` (VM-isolated managed worker runtime), `iii-init` (PID 1 for microVM workers),
`iii-filesystem` and `iii-network` (sandbox filesystem and userspace TCP/IP), `iii-supervisor` and
`iii-shell-proto` / `iii-shell-client` (in-VM process supervision and the shell-exec channel),
`scaffolder-core` (project templates), `iii-clap-docs` (renders the CLI reference from clap),
`motia-tools`.

## Workspaces and dependency resolution

- **Rust** — one Cargo workspace; members are listed in the root `Cargo.toml`. Internal crates use
  `{ workspace = true }`; Cargo substitutes real versions when publishing.
- **JS/TS** — pnpm workspaces (`pnpm-workspace.yaml`) with Turborepo (`turbo.json`) for build
  orchestration. Internal references use `workspace:*` / `workspace:^`.
- **Python** — `uv`, with local packages wired through `[tool.uv.sources]` editable installs.

Use **`pnpm`, never `npm`**. Node >= 20, pnpm 10.19.0 (`packageManager` field), Python >= 3.10.

## Commands

The `Makefile` is the primary entry point: its targets mirror the CI jobs, so a green `make` locally
means a green CI. Prefer it over ad-hoc commands.

### Setup

```bash
make install          # pnpm install --frozen-lockfile + uv sync for the Python SDK
make install-hooks    # pre-commit hook that runs cargo fmt --check (recommended)
```

### The loop you will use most

```bash
make fix              # cargo fmt --all + ruff --fix + clippy --fix + biome lint:fix
make check            # lint + fmt-check + typecheck + build SDK/console
make ci-local         # everything CI runs: engine, all four SDKs, console
```

### Per area

```bash
make ci-engine        # engine-build + engine-test + fmt check
make engine-test      # installs iii-worker to PATH first, then runs the engine suites
make ci-sdk-node      # starts an engine with bridges, typechecks, builds, runs vitest
make ci-sdk-python    # ruff + mypy + pytest against a live engine
make ci-sdk-rust      # fmt + clippy + cargo test -p iii-sdk against a live engine
make ci-console       # lint frontend + build frontend and the Rust binary
make cli-docs         # regenerate docs/next/cli-reference/ (CI fails if this is stale)
```

### Underlying tools, when you need something narrower

```bash
cargo build -p iii --all-features        # engine (debug, what CI builds)
cargo test -p iii --all-features         # engine tests, incl. doctests
cargo test -p iii-sdk --all-features     # Rust SDK
cargo fmt --all && cargo clippy --workspace --all-targets --all-features -- -D warnings

pnpm build                                # all JS/TS via Turborepo
pnpm test:sdk-node                        # Node SDK only
pnpm fmt                                  # biome format --write . && cargo fmt --all
pnpm fmt:check                            # what CI checks
pnpm dev:console | dev:docs | dev:website # dev servers

cd sdk/packages/python/iii && uv run pytest
cd sdk/packages/go/iii && go test ./...
```

### Running an engine

```bash
cargo run --release           # reads ./config.yaml (or pass --config <path>)
iii --use-default-config      # built-in modules + in-memory OTel, no config file needed
iii console                   # open the developer console

make engine-up                # test engine on the test ports, for SDK integration tests
make engine-up-bridges        # test engine + bridge backend + bridge (needed by Node SDK tests)
make engine-down              # stop them all
```

| Context                        | WebSocket | HTTP   | Stream | Metrics |
| ------------------------------ | --------- | ------ | ------ | ------- |
| Default runtime                | `49134`   | `3111` | `3112` | `9464`  |
| Test engine (`make engine-up`) | `49199`   | `3199` | —      | —       |

SDK tests read `III_URL` and `III_HTTP_URL`; the `make` targets set them for you.

## Engine architecture

```
engine/src/
  main.rs, lib.rs        entry point and library root
  protocol.rs            the WebSocket wire protocol shared with every SDK
  trigger.rs             trigger model; trigger_formats.rs for wire shapes
  function.rs            function model
  engine/                core runtime
  invocation/            invocation paths: HTTP function/invoker, auth, signatures, URL validation
  worker_connections/    connection lifecycle and traits
  workers/               built-in workers (below)
  builtins/              in-process kv, queue, pubsub-lite backends
  config/                config loading
  cli/                   the `iii` CLI (project, registry, exec, update, telemetry, gen-docs).
                         `console`, `cloud`, and `worker` are passthrough stubs that dispatch to
                         external binaries resolved in `cli/registry.rs`
  telemetry.rs           usage telemetry (III_TELEMETRY_ENABLED=false in tests and CI)
```

**Built-in workers** live in `engine/src/workers/` — `queue`, `state`, `stream`, `pubsub`, `cron`,
`rest_api`/`http_functions`, `observability`, `telemetry`, `shell`, `configuration`, `engine_fn`,
`bridge_client`, `worker`. Most have their own README with the exact `iii worker add` and skill
install commands. In `config.yaml` they are addressed by registry name (`iii-queue`, `iii-state`,
`iii-stream`, `iii-pubsub`, `iii-cron`, `iii-http`, `iii-observability`, `iii-bridge`, `iii-exec`,
`configuration`), each with an `adapter` block choosing its backend (`kv`, `redis`, `local`,
`rabbitmq`, …).

**Namespaces** are a runtime routing dimension: registrations and invocations carry an optional
`namespace`, absent means the engine default. The semantics — blank-vs-absent, inheritance from the
registering connection, strict addressing — are documented in `engine/src/protocol.rs`; read it
before touching namespace-adjacent code.

## Conventions

### Always

- `pnpm`, never `npm`.
- Function IDs use `::`: `orders::validate`, `reports::daily-summary`.
- HTTP `api_path` values take a leading slash: `/orders`, `/users/:id`.
- Cron trigger configs use `expression`, **not** `cron`, and the expression is 7-field
  (`sec min hour dom month dow year`): `"0 0 9 * * * *"`.
- `cargo fmt --all` before committing Rust; `pnpm fmt` before committing JS/TS.
- Internal pnpm deps use `workspace:*`.
- Rust sources under the paths listed in `engine/licenserc.toml` need the license header from
  `engine/header.txt`. `license-check.yml` enforces it.
- Every `SKILL.md` has `## When to Use` and `## Boundaries` sections, and its `name` field matches
  its directory name exactly. SkillKit validates both.

### Ask first

- Public SDK API surface (npm / PyPI / crates.io / Go module).
- The engine config schema (`engine/config.yaml`).
- The WebSocket protocol between SDKs and engine.
- New engine modules or built-in workers.
- CI/CD workflows under `.github/`.

### Never

- Commit secrets, API keys, or credentials.
- Push directly to `main`.
- Change engine licensing (ELv2) or SDK licensing (Apache-2.0).
- Hand-edit `docs/*/cli-reference/` or `docs/docs.json` formatting — both are generated and
  `.prettierignore`d; regenerate instead.
- Bump one manifest's version in isolation.

### Formatting

- **Biome** (`biome.json`) formats and lints `.ts/.tsx/.js/.jsx/.json/.jsonc` only — 2-space indent,
  120 columns, LF. It does not touch Markdown, Rust, or Python.
- **Markdown** follows `.prettierrc`: `proseWrap: always`, `printWidth: 100`.
- **Rust**: `cargo fmt --all`, defaults. **Python**: ruff, line length 120, mypy `strict`.

## Code style

```rust
// Rust SDK — :: function IDs, leading-slash paths, `expression` for cron
iii.register_function(
    RegisterFunction::new("orders::validate", validate_order)
        .description("Validate an incoming order"),
);
iii.register_trigger(
    IIITrigger::Http(HttpTriggerConfig::new("/orders/validate").method(HttpMethod::Post))
        .for_function("orders::validate"),
);
iii.register_trigger(
    IIITrigger::Cron(CronTriggerConfig::new("0 0 9 * * * *"))
        .for_function("reports::daily-summary"),
);
```

```typescript
// TypeScript SDK
iii.registerTrigger({
  type: 'http',
  function_id: 'orders::validate',
  config: {
    api_path: '/orders/validate',
    http_method: 'POST',
    middleware_function_ids: ['middleware::auth', 'middleware::rate-limit'],
  },
})

iii.registerTrigger({
  type: 'cron',
  function_id: 'reports::daily-summary',
  config: { expression: '0 0 9 * * * *' },
  metadata: { owner: 'billing-team', priority: 'high' },
})
```

```python
# Python SDK — same rules
iii.register_trigger({
    "type": "http",
    "function_id": "orders::validate",
    "config": {"api_path": "/orders/validate", "http_method": "POST"},
})
```

## Testing

- **Engine** — `make engine-test`. It builds `iii-worker` and installs it to `~/.local/bin` first,
  because the engine integration tests shell out to that binary; running `cargo test -p iii` without
  it will fail confusingly. The VM crates (`iii-worker`, `iii-filesystem`, `iii-network`,
  `iii-init`) run with **default features** — `--all-features` pulls in the KVM-only
  `integration-vm`/`-oci` suites, which have their own jobs. `iii` itself runs with
  `--all-features`.
- Engine tests live in `engine/tests/` (largely `*_e2e.rs` integration suites). Several rely on
  shared in-process global state (tracing/metrics singletons, ports), so process-per-test runners
  like nextest are not used; keep new tests isolation-aware.
- **RabbitMQ** suites need a broker and run as their own CI job:
  `cargo test -p iii --test rabbitmq_queue_integration` with `RABBITMQ_URL` set.
- **SDK tests need a live engine.** Start it with `make engine-up` (Node SDK also needs the bridges:
  `make engine-up-bridges`), then `make test-sdk-node|-python|-rust`, then `make engine-down`. The
  `ci-sdk-*` targets do the whole dance for you.
- **Node SDK** uses vitest with v8 coverage thresholds at 60% (lines/functions/branches/statements);
  dropping below fails the run.
- **Coverage** — `make coverage` reproduces CI's `cargo llvm-cov` run.

`III_TELEMETRY_ENABLED=false` is exported by the Makefile and set in CI; keep it off in tests.

## CI gates

`.github/workflows/ci.yml` runs on pushes and PRs to `main`. It has `paths-ignore` for `docs/**`,
`skills/**`, `website/**`, `.cursor/**`, and all `*.md`/`*.mdx`, so documentation-only changes skip
the heavy jobs entirely.

Jobs that will fail a PR:

| Job                            | What it checks                                                           |
| ------------------------------ | ------------------------------------------------------------------------ |
| `engine-build`                 | `cargo build -p iii --all-features`; publishes the binary other jobs use |
| `engine-coverage`              | The full engine + crate suite under `cargo llvm-cov`                     |
| `engine-fmt`                   | `cargo fmt --all -- --check`                                             |
| `cli-docs-built`               | Regenerates `docs/next/cli-reference/` and fails on any diff             |
| `engine-build-matrix`          | Builds on macOS, Windows, Linux gnu + musl                               |
| `worker-build-matrix`          | Builds `iii-worker` on the release targets                               |
| `sdk-{node,python,rust,go}-ci` | Lint, typecheck, build, and test each SDK against a live engine          |
| `console-ci`                   | Frontend lint + build, plus the Rust console binary                      |
| `engine-rabbitmq-tests`        | The RabbitMQ queue integration suite against a broker service            |
| `worker-test-{vm,oci}-*`       | The KVM/OCI sandbox suites for `iii-worker`                              |

Two more workflows gate a PR alongside `ci.yml`: `license-check.yml` (license headers on engine
sources, driven by `engine/licenserc.toml`) and `checklist-checker.yml` (the contributor
license-agreement checkbox).

If you change the CLI's clap definitions, run `make cli-docs` and commit the result — that is the
single most common avoidable CI failure.

## Docs

Docs are versioned Next / Latest / archive:

- **Author under `docs/next/`.** That folder is what a stable release promotes.
- **Latest** is the `docs/` root (unprefixed paths). **Archives** are `docs/MAJOR-MINOR-0/`
  (e.g. `docs/0-21-0/`). `docs/changelog/` is shared and never moves.
- Generated trees must be regenerated before a release: `make cli-docs` for the CLI reference and
  `pnpm tsx docs/next/scripts/generate-api-docs.mts` (or `pnpm generate:api-docs`) for the API
  reference.
- Full mechanics: `docs/RELEASING.md` and the `create-tag.yml` section of
  `.github/workflows/WORKFLOWS.md`.

Writing style follows the Divio system (tutorial / how-to / reference / explanation) — the rules and
per-quadrant templates are in `.cursor/rules/docs.mdc` and `.cursor/skills/`.

`skills/` holds six knowledge skills (`iii-getting-started`, `iii-core-primitives`,
`iii-sdk-reference`, `iii-engine-config`, `iii-architecture-patterns`, `iii-error-handling`) plus
the `presentation` tool skill. Worker-specific capability skills deliberately live with their
workers in `iii-hq/workers`, not here — do not duplicate queue/state/cron material into `skills/`.

## Releases

Releases are workflow-driven; never tag by hand.

1. `create-tag.yml` (manual dispatch, `main` only) bumps every manifest in lockstep, rotates docs on
   stable releases, and pushes the annotated tag `iii/v{version}`.
2. The tag push triggers `release-iii.yml`: GitHub Release, engine + worker + init binaries, Docker
   image, console, and all four SDK registries, then the built-in workers and their skills.
3. `alpha-release.yml` publishes an `iii-alpha/v*` prerelease from any feature branch without
   touching `main` — the isolated way to test a branch end to end.

## Licensing

`engine/` is Elastic License 2.0. Everything else — SDKs, CLI, console, docs, website — is
Apache-2.0. **All contributions, including to the engine, are made under Apache-2.0**; see
`CONTRIBUTING.md`.
