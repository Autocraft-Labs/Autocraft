<div align="center">
  <img src=".github/assets/banner.svg" alt="Autocraft" width="100%" />
</div>

<div align="center">

**A decentralized quest-based learning platform on the Stellar blockchain**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/Rust-1.80.0-orange)](rust-toolchain.toml)
[![Soroban SDK](https://img.shields.io/badge/soroban--sdk-22-blue)](Cargo.toml)
[![Node](https://img.shields.io/badge/Node-24.12.0-green)](frontend/package.json)

</div>

---

## Overview

Autocraft is a serverless, on-chain learning platform where quest owners create structured learning paths with milestones and token rewards. Learners complete quests, earn USDC, and receive NFT certificates — all enforced by Soroban smart contracts on Stellar. There is no backend server; the Stellar blockchain is the source of truth.

<div align="center">
  <img src=".github/assets/how-it-works.svg" alt="How It Works" width="80%" />
</div>

---

## Repository Structure

```
Autocraft/
├── contracts/
│   ├── common/          # Shared types, constants, address validation
│   ├── quest/           # Quest creation and learner enrollment
│   ├── milestone/       # Milestone definition and completion verification
│   ├── rewards/         # Token pool management and reward distribution
│   ├── certificate/     # NFT certificates for quest completion
│   └── testutils/       # Shared test helpers
├── frontend/            # React 19 + Vite + Tailwind CSS v4
├── services/
│   └── event-indexer/   # Soroban event → PostgreSQL subscriber
├── docs/                # Architecture, API reference, ADRs, runbooks
└── scripts/             # Contract binding generation, doc checks
```

---

## Architecture

<div align="center">
  <img src=".github/assets/architecture.svg" alt="Architecture" width="80%" />
</div>

The platform has four Soroban smart contracts with clearly separated responsibilities:

| Contract | Responsibility |
|:---------|:--------------|
| `quest` | Quest creation, metadata, learner enrollment |
| `milestone` | Milestone definitions, completion verification (owner or peer review) |
| `rewards` | Token pool funding and reward distribution |
| `certificate` | ERC-721-style NFT minting on quest completion |

The frontend (React) is the orchestration layer — it sequences contract calls and signs every transaction via the [Freighter](https://freighter.app) wallet. No centralized backend.

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for full sequence diagrams and the storage model.

---

## Smart Contracts

### Prerequisites

- Rust `1.80.0` (managed via `rust-toolchain.toml`)
- `wasm32-unknown-unknown` target
- Stellar CLI v25.1.0+

```bash
rustup target add wasm32-unknown-unknown
cargo install --locked stellar-cli --version 25.1.0
```

### Build

```bash
cargo build --workspace
stellar contract build
```

WASM outputs land in `target/wasm32v1-none/release/`.

### Test

```bash
cargo test --workspace
```

### Contracts Overview

**`quest`** — Entry point. Owners call `create_quest(owner, name, description, category, tags, token_addr, visibility, max_enrollees)`. Learners self-enroll on public quests via `join_quest`, or owners enroll them via `add_enrollee`. Tracks `OwnerQuests` and `EnrolleeQuests` indexes for efficient lookups.

**`milestone`** — Owners call `create_milestone(owner, quest_id, title, description, reward_amount, requires_previous)`. Supports two verification modes: `OwnerOnly` (owner calls `verify_completion`) and `PeerReview(n)` (n peer approvals auto-complete a milestone). Automatically mints a certificate when all milestones are complete.

**`rewards`** — Initialized once with a SAC token address and references to `quest` and `milestone` contracts. Quest owners fund pools via `fund_quest`; the frontend triggers `distribute_reward` after milestone verification. Supports `refund_pool` after a quest is archived (7-day grace period).

**`certificate`** — Mints non-fungible tokens using Stellar's NFT standard. Stores metadata: quest ID, quest name, category, completion date, issuer, and recipient. Token IDs are monotonically increasing per contract.

**`common`** — Shared types (`QuestInfo`, `QuestStatus`, `Visibility`) and TTL constants (`BUMP = 518,400 ledgers ≈ 30 days`). No deployed contract — imported as a library crate.

See [docs/CONTRACTS_OVERVIEW.md](docs/CONTRACTS_OVERVIEW.md) for full state tables and events.

### Deploying to Testnet

See [docs/deploy-testnet.md](docs/deploy-testnet.md) for a full step-by-step runbook.

Quick summary:

```bash
# 1. Deploy contracts
stellar contract deploy --wasm target/wasm32v1-none/release/rewards.wasm \
  --source-account <DEPLOYER> --network testnet --alias autocraft-rewards-testnet

stellar contract deploy --wasm target/wasm32v1-none/release/quest.wasm \
  --source-account <DEPLOYER> --network testnet --alias autocraft-quest-testnet

stellar contract deploy --wasm target/wasm32v1-none/release/milestone.wasm \
  --source-account <DEPLOYER> --network testnet --alias autocraft-milestone-testnet

# 2. Initialize rewards
stellar contract invoke --id <REWARDS_ID> --source-account <DEPLOYER> \
  --network testnet -- initialize --token_addr <TOKEN_ID>
```

Supported networks are configured in [`environments.toml`](environments.toml): `development`, `testnet`, `mainnet`.

---

## Frontend

React 19 · TypeScript 5.9 · Vite 8 · Tailwind CSS v4 · shadcn/ui · Freighter wallet

### Setup

```bash
cd frontend
pnpm install
cp .env.example .env   # fill in contract IDs
pnpm dev               # http://localhost:5173
```

### Scripts

| Command | Description |
|:--------|:------------|
| `pnpm dev` | Dev server with HMR |
| `pnpm build` | Type-check + production build |
| `pnpm test` | Unit tests (Vitest) |
| `pnpm test:e2e` | End-to-end tests (Playwright) |
| `pnpm lint` | ESLint |
| `pnpm format` | Prettier |
| `pnpm generate:bindings` | Regenerate contract TypeScript clients |

### Environment Variables

Copy `frontend/.env.example` and fill in your deployed contract IDs and network settings before running `pnpm dev` or `pnpm build`.

### Key Pages

| Route | Description |
|:------|:------------|
| `/` | Landing — hero, how-it-works, feature highlights |
| `/dashboard` | Active quests list with stats |
| `/quest/:id` | Quest detail — milestones, enrollees, progress |
| `/create` | Quest creation wizard |
| `/profile` | User earnings, certificates, quest history |
| `/leaderboard` | Top earners and quest creators |
| `/creator` | Creator analytics dashboard |

### Wallet Integration

Uses [Freighter](https://freighter.app) (`@stellar/freighter-api`). Switch Freighter to **Testnet** before connecting on a local/testnet environment.

### Design System

Neo-brutalist: `#FACC15` yellow + `#000000` black + `#FFFFFF` white. 2–3 px solid black borders, solid-offset shadows, no blur. See [`frontend/src/index.css`](frontend/src/index.css) for design tokens.

---

## Event Indexer

A TypeScript service that polls Stellar RPC for Soroban contract events and persists them to PostgreSQL for analytics dashboards and fraud detection.

### Setup

```bash
cd services/event-indexer
cp .env.example .env   # DATABASE_URL, contract IDs, SENTRY_DSN
npm install
npm run db:migrate
npm start
```

### Key Environment Variables

| Variable | Required | Description |
|:---------|:--------:|:------------|
| `DATABASE_URL` | ✓ | PostgreSQL connection string |
| `QUEST_CONTRACT_ID` | ✓ | Deployed quest contract address |
| `MILESTONE_CONTRACT_ID` | ✓ | Deployed milestone contract address |
| `REWARDS_CONTRACT_ID` | ✓ | Deployed rewards contract address |
| `SOROBAN_RPC_URL` | | Defaults to mainnet RPC |
| `POLL_INTERVAL_MS` | | Defaults to 5000 ms |

### Dashboard Views

```sql
SELECT * FROM v_quest_activity LIMIT 30;
SELECT * FROM v_reward_activity LIMIT 30;
SELECT * FROM v_milestone_completions LIMIT 30;
```

See [services/event-indexer/README.md](services/event-indexer/README.md) and [docs/operations/event-indexer-runbook.md](docs/operations/event-indexer-runbook.md).

---

## Documentation

| Document | Description |
|:---------|:------------|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | System overview, sequence diagrams, storage model |
| [docs/CONTRACTS_OVERVIEW.md](docs/CONTRACTS_OVERVIEW.md) | Contract-by-contract reference with state and events |
| [docs/api-reference.md](docs/api-reference.md) | Every public function with signatures and error codes |
| [docs/EVENT_REFERENCE.md](docs/EVENT_REFERENCE.md) | Every emitted event with topics and payload |
| [docs/INTEGRATION_TESTING.md](docs/INTEGRATION_TESTING.md) | Local node setup and smoke test walkthrough |
| [docs/deploy-testnet.md](docs/deploy-testnet.md) | Testnet deployment runbook |
| [docs/reward-economics.md](docs/reward-economics.md) | Token pool economics and distribution models |
| [docs/security-audit.md](docs/security-audit.md) | Security audit findings and mitigations |
| [docs/adr/](docs/adr/) | Architecture Decision Records |

---

## Contributing

1. Fork the repo and create a branch from `main`.
2. Run `cargo test --workspace` for contract changes; `pnpm test` for frontend changes.
3. Follow the existing code style (Rust: `cargo clippy`; TypeScript: `pnpm lint && pnpm format`).
4. Open a pull request — the template at [`.github/PULL_REQUEST_TEMPLATE.md`](.github/PULL_REQUEST_TEMPLATE.md) will guide you.

Bug reports and feature requests: use the [issue templates](.github/ISSUE_TEMPLATE/).

---

## License

[MIT](LICENSE) © 2026 Autocraft
