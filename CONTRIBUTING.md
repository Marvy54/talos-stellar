# Contributing to Talos Protocol

Thanks for helping improve Talos Protocol. This guide covers the local development setup for the Node.js, Python, and Rust parts of the monorepo, the required environment files, and the workflow we expect for pull requests.

If you are looking for a good place to start, browse the open Wave issues:
https://github.com/enliven17/talos-stellar/issues?q=iss%3Aissue+iss%3Aopen+label%3A%22Stellar+Wave%22

## Prerequisites

Install these before you start working locally:

- Node.js 20 or newer
- pnpm 9 or newer
- Python 3.11 or newer
- `uv`
- Rust stable toolchain and `cargo`
- Soroban CLI, installed as `stellar` via `cargo install --locked stellar-cli --features opt`
- [`gitleaks`](https://github.com/gitleaks/gitleaks) secret scanner — `brew install gitleaks`, or download a binary from the [releases page](https://github.com/gitleaks/gitleaks/releases) (required for `pnpm run secrets:check`)

For the Rust contracts, also add the Wasm target:

```bash
rustup target add wasm32-unknown-unknown
```

## Devcontainer (Codespaces / VS Code)

The repo ships a `.devcontainer/devcontainer.json` that installs the full toolchain so you can start working in one click. It includes:

- Node 20 and pnpm 9
- Python 3.11 with `uv`
- Rust stable with the `wasm32-unknown-unknown` target
- `stellar-cli`` installed as `stellar`
- `gitleaks` for `pnpm run secrets:check`

The container forwards the ports used by `docker-compose.yml` and the Next.js dev server (see `.devcontainer/devcontainer.json` for the authoritative list). On creation it runs `pnpm install`, `uv sync --extra dev` in `packages/prime-agent`, and prepares the Rust target.

To use it:

1. Open the repo in GitHub Codespaces, or in VS Code run **Dev Containers: Reopen in Container**.
2. Wait for the post-create command to finish installing workspace dependencies.
3. Verify the environment:

```bash
pnpm --dir web test:unit
cd packages/prime-agent && uv run pytest
cd contracts && cargo test
```

The first build of the devcontainer image typically takes **10-20 minutes** on a cold Codespaces machine (it compiles `stellar-cli` from source). Subsequent starts reuse the cached image and take under a minute. If you prefer a local setup, follow the instructions below.

## Repository Layout

- `web/` - Next.js application, API routes, and frontend
- `packages/prime-agent/` - Python agent runtime
- `contracts/` - Soroban smart contracts and deploy scripts (see `contracts/EVENTS.md` for the event schema)
- `packages/openclaw/` - skill definitions and agent helper code

## Setup

### 1. Clone the repo

```bash
git clone https://github.com/enliven17/talos-stellar.git
cd talos-stellar
```

### 2. Install Node.js dependencies

Install the workspace dependencies from the repository root:

```bash
pnpm install
```
Generate the dependency license report from the installed workspace graph:

```bash
pnpm licenses:report
```

The command writes `dist/licenses/dependency-licenses.json` and
`dist/licenses/dependency-licenses.md`. It fails closed when pnpm cannot read
the dependency graph or any dependency has missing or ambiguous license
metadata. The generated directory is a CI artifact and should not be committed.

If you only need the web app, you can still work from the root with `pnpm dev` because the root package forwards to `web/`.

### 3. Set up Python with `uv`

Install the agent dependencies and the dev extras:

```bash
cd packages/prime-agent
uv sync --extra dev
```

Run the agent with:

```bash
uv run talos-agent start
```

### 4. Set up Rust and Soroban

From `contracts/`:

```bash
cargo test
cargo build --target wasm32-unknown-unknown --release
```

To deploy the contracts to testnet, use the provided script from a Bash-compatible shell:

```bash
./deploy.sh testnet
[```

The script prints the deployed contract IDs that need to be copied into the web app environment file.

## Environment Files

The repo already includes verified example files for each runtime. Copy them before running locally:

- `web/.env.example` -> `web/.env.local`
- `packages/prime-agent/.env.example` -> `packages/prime-agent/.env`
- `contracts/.env.example` -> `contracts/.env`

### Web app env

`web/.env.example` contains the web app, database, Stellar, x402, and AI service variables. The most important entries are:

- `DATABASE_URL` and `DIRECT_URL`
- `STELLAR_NETWORK`, `STELLAR_HORIZON_URL`, and `STELLAR_RPC_URL`
- `STELLAR_OPERATOR_SECRET_KEY` and `STELLAR_OPERATOR_PUBLIC_KEY`
​
- `NEXT_PUBLIC_STELLAR_OPERATOR_PUBLIC_KEY`
​
- `NEXT_PUBLIC_TALOS_REGISTRY_CONTRACT` and `NEXT_PUBLIC_TALOS_NAME_SERVICE_CONTRACT`
​
- `X402_FACILITATOR_URL`, `X402_API_KEY` and `X402_SETTLEMENT_NETWORK​`
- `GROQ_API_KEY` or `OPENAI_API_KEY`
- `TAVILY_API_KEY`

If you deploy new contracts, update the contract IDs in `web/.env.local` with the values printed by `contracts/deploy.sh`.

### Prime agent env

`packages/prime-agent/.env.example` configures the Talos agent runtime. It documents:

- `TALOS_API_KEY` or `TALOS_API_KEYS`
- `TALOS_ID` and `TALOS_API_URL`
- `GROQ_API_KEY` or `OPENAI_API_KEY`
- `X_USERNAME`, `X_PASSWORD`, and `X_EMAIL`
- `BROWSER_HEADLESS`
- agent timing and approval settings such as `AGENT_CYCLE_INTERVAL`, `POLLING_INTERVAL`, and `APPROVAL_THRESHOLD`
- opt-in durable job inbox/outbox settings under `TALOS_DURABLE_JOB_EFFECTS_*` and `TALOS_JOB_EFFECT_*`

### Contracts env

`contracts/.env.example` is for contract deployment and signer configuration. It includes:

- `STELLAR_SECRET_KEY`
- `TALOS_PROTOCOL_WALLET`
- commented placeholders for the post-deployment contract IDs used by the web app

## Running the Project

### One-command local integration stack

A reproducible local integration stack is available for contributors:

```bash
pnpm stack:up
```

This starts PostgreSQL, a mock Stellar provider, the web app, and seeds the local database. The web service exposes health and readiness endpoints at `/api/health` and `/api/health/ready`.

Use these commands to manage the environment:

```bash
pnpm stack:logs
pnpm stack:down
pnpm stack:reset
```

`pnpm stack:reset` destroys the stack volumes and re-imports everything. To clear local test data without recreating the stack, use the safe reset instead:

```bash
pnpm stack:reset-data --dry-run    # preview the tables and row counts
pnpm stack:reset-data --yes        # truncate local test data
pnpm stack:reset-data --yes --seed # truncate, then re-run db:seed
```

It only ever targets a loopback database, requires `--yes` to destroy anything,
and never logs credentials. See
[docs/local-test-data-reset.md](./docs/local-test-data-reset.md) for the full
behavior contract. Regression tests for the shell layer: `bash scripts/local-stack.test.sh`.

The stack defaults to the web service and a mock Stellar provider. Add the optional prime-agent service with:

```bash
docker compose --profile agent up -d prime-agent
```

#### Deterministic smoke fixture

The local stack's expected shape — ports, health endpoints, and mock marketplace
agents — is captured in a deterministic fixture at
`web/tests/fixtures/smoke-fixture.json`. Regenerating it is byte-stable (no
timestamps, randomness, or network): every run on any machine produces identical
output, so diffs in review are meaningful.

The exact local command to verify the fixture is canonical and drift-free:

```bash
pnpm smoke:fixture:check
```

To regenerate it after intentionally changing the fixture definition in
`scripts/smoke-fixture.lib.mjs`:

```bash
pnpm smoke:fixture:gen
```

Focused tests (positive, negative, boundary, regression, and privacy coverage):

```bash
pnpm --dir web exec vitest run tests/smoke-fixture.unit.test.ts
```

Behavior notes:

- **Fail closed** — ambiguous, malformed, or secret-shaped input aborts with a
  non-zero exit and an explicit, privacy-safe error. Field names that look like
  secrets (`secret`, `seed`, `private key`, `password`, `api key`, `proof`, …)
  are rejected outright and their values are never logged or returned.
- **Placeholder data only** — the fixture never contains real keys, seeds, or
  payment proofs; agent addresses are deterministic truncated placeholders in
  the same style as the demo seed.
- **CI** — `pnpm smoke:fixture:check` is safe to wire into any workflow step;
  it exits `0` when the committed fixture matches the canonical output and `1`
  with a regeneration hint otherwise.

### Web

From the repo root:

```bash
pnpm dev
```

Or from `web/`:

```bash
cd web
pnpm dev
```

Other useful web commands:

```bash
pnpm build
pnpm lint
pnpm test:unit
pnpm test:e2e
pnpm test:bench       # Run performance benchmarks
```

See [BENCHMARKS.md](./BENCHMARKS.md) for the benchmark system documentation.

Any PR that changes `web/drizzle/**` or `web/src/db/**` is validated by the `Web Migrations CI`
workflow, which applies your migrations to an ephemeral Postgres instance. See
[`MIGRATIONS.md`](./MIGRATIONS.md) for what it checks, how to reproduce it locally, and rollback
guidance.

### Prime Agent

```bash
cd packages/prime-agent
uv run talos-agent start
```

### Contracts

```bash
cd contracts
cargo test
cargo build --target wasm32-unknown-unknown --release
```

If you are iterating on contract behavior, also run the Wasm-target test path used by CI:

```bash
cargo test --target wasm32-unknown-unknown
```

## Focused Test Selection and CI Triage

Run the smallest test set that covers the files you touched before falling back to a full suite. The commands below avoid database resets and hidden local state. Run them after installing dependencies with `pnpm install` at the repository root, `uv sync --extra dev` in `packages/prime-agent/`, or the Rust setup from the prerequisites. In this section, commands that call package scripts use `pnpm run <script>` so the script name can be checked directly in the package's `package.json`.

### Web changes

Use the web package scripts for Next.js, API, database, and DevX changes. The main test artifact is the Vitest report in stdout. Benchmark jobs also upload `.benchmarks/` as the `benchmark-artifacts` workflow artifact.

```bash
# POSIX shells
pnpm --dir web exec vitest run tests/health.test.ts
pnpm --dir web exec vitest run tests/*.unit.test.ts
pnpm --dir web exec vitest run tests/openapi-snapshot.test.ts
pnpm --dir web exec vitest run src/area/devx/__tests__/runner.test.ts
pnpm --dir web run lint
pnpm --dir web exec tsc --noEmit
```

```powershell
# Windows PowerShell
pnpm --dir web exec vitest run tests\health.test.ts
pnpm --dir web exec vitest run tests\*.unit.test.ts
pnpm --dir web exec vitest run tests\openapi-snapshot.test.ts
pnpm --dir web exec vitest run src\area\devx\__tests__\runner.test.ts
pnpm --dir web run lint
pnpm --dir web exec tsc --noEmit
```

Choose the focused command by area:

| Changed files | Focused command | CI workflow |
| --- | --- | --- |
| `web/src/app/api/**`, `web/src/lib/openapi.ts`, `web/tests/fixtures/openapi.snapshot.json` | `pnpm --dir web exec vitest run tests/openapi-snapshot.test.ts` | `Web OpenAPI CI` |
| `web/drizzle/**`, `web/src/db/**`, `web/drizzle.config.ts` | `pnpm --dir web run db:migrate`, then the specific DB test with `pnpm --dir web exec vitest run tests/<name>.test.ts` | `Web Migrations CI` |
| `web/src/area/devx/**` | `pnpm --dir web exec vitest run src/area/devx/__tests__/runner.test.ts` | `Benchmark CI - regression gates` |
| API route or library unit tests | `pnpm --dir web exec vitest run tests/<name>.test.ts` | `Deploy Web -> Vercel` |
| Local test-data reset (`web/src/lib/local-test-data-reset.ts`, `web/src/db/reset-test-data.ts`, `scripts/local-stack.sh`) | `pnpm --dir web exec vitest run tests/local-test-data-reset.unit.test.ts; bash scripts/local-stack.test.sh` | `Deploy Web -> Vercel` |
| Backup or restore paths | `pnpm --dir web exec vitest run tests/backup-restore-fixture.test.ts tests/backup-crypto.test.ts tests/backup-types.test.ts` | `Web Backups CI` |

Use `pnpm --dir web run test:e2` only when API route behavior depends on the running app or cross-route state. Use the local stack with `pnpm stack:up` when you need Postgres plus the mock Stellar provider, and clean it up with `pnpm stack:down`. Do not use `pnpm stack:reset` unless you intentionally want to destroy and recreate local stack data; `pnpm stack:reset-data` clears test rows without touching the stack volumes.

### SDK changes

The TypeScript SDK lives in `packages/sdk/`. The expected build artifact is `packages/sdk/dist/`, including the browser bundle at `packages/sdk/dist/browser/sdk.bundle.js`.

```bash
# POSIX shells
pnpm --filter @talos-protocol/sdk exec vitest run tests/client.test.ts
pnpm --filter @talos-protocol/sdk run build
pnpm --filter @talos-protocol/sdk exec tsc --noEmit
```

```powershell
# Windows PowerShell
pnpm --filter @talos-protocol/sdk exec vitest run tests\client.test.ts
pnpm --filter @talos-protocol/sdk run build
pnpm --filter @talos-protocol/sdk exec tsc --noEmit
```

Choose the focused command by area:

| Changed files | Focused command | CI workflow |
| --- | --- | --- |
| `packages/sdk/src/**`, `packages/sdk/tests/**` | `pnpm --filter @talos-protocol/sdk exec vitest run tests/<name>.test.ts` | `SKD Compatibility Tests` |
| `packages/sdk/package.json`, `packages/sdk/tsup.config.ts` | `pnpm --filter @talos-protocol/sdk run build` | `SGD Compatibility Tests` |
