# Rotula Soroban Contracts — Savings rules on Stellar

> Soroban contract experiments for transparent community savings groups.

[![CI](https://github.com/Rotula-Labs/rotula-contracts/actions/workflows/rust.yml/badge.svg?branch=main)](https://github.com/Rotula-Labs/rotula-contracts/actions/workflows/rust.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Stellar](https://img.shields.io/badge/Stellar-Soroban-%237b2ff7?logo=stellar)](https://developers.stellar.org)
[![Rust](https://img.shields.io/badge/Rust-stable-%23000000?logo=rust)](https://www.rust-lang.org)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

Rotula is being built for communities that already save together through Ajo, Esusu, and other rotating savings circles. This repository contains the Rust Soroban contract that models the on-chain side of that idea: a group, its members, a configured Stellar token, contribution state, and the rules governing payouts or goal-based withdrawals.

Stellar is Rotula's intended settlement network. Soroban makes it possible to represent group rules as contract logic and to emit events as state changes. The contract does not replace the WhatsApp experience or Rotula's backend; those systems must still create groups, coordinate members, authorize invocations, submit transactions, and communicate confirmed outcomes.

**Status:** a version of the contract code is deployed to Stellar Testnet for review. No savings-group instance has been initialized, and no token has been deposited. The backend integration is still being aligned with the current contract interface. This is not a production-ready custody system; do not use real funds.

## How it uses Stellar

This repository is the on-chain half of Rotula, and everything in it is Stellar-native:

- **Soroban smart contract** compiled to `wasm32v1-none` (SDK 27) and deployed to Stellar Testnet.
- **SEP-41 token movement** through a configured Stellar token contract; USDC on Stellar is the intended asset, but the token address is an initialization argument rather than a hard-coded constant.
- **Soroban authorization** (`require_auth`) gates every membership change, contribution, payout, and withdrawal; there is no off-chain signature that can move pooled funds.
- **Contract events** for initialization, membership, contributions, payouts, withdrawals, pause changes, and cycle operations, readable by any Stellar RPC client.
- **Soroban storage TTL** is extended for group and member state so long-running groups do not archive mid-cycle.

## Table of Contents

- [How it uses Stellar](#how-it-uses-stellar)
- [Testnet deployment](#testnet-deployment)
- [Why Soroban for community savings](#why-soroban-for-community-savings)
- [Contract behavior](#contract-behavior)
- [Interface](#interface)
- [Rotational cycle notes](#rotational-cycle-notes)
- [Prerequisites](#prerequisites)
- [Build and test](#build-and-test)
- [Integration with Rotula](#integration-with-rotula)
- [Security and network use](#security-and-network-use)
- [Contributing](#contributing)
- [License](#license)

### Testnet deployment

- **Network:** Stellar Testnet
- **Contract ID:** [`CBQCB6KIXAPDPKGNZFR3KSBBJNEJRFN3MAQL7S46JGFA5I22AUL4OPYL`](https://stellar.expert/explorer/testnet/contract/CBQCB6KIXAPDPKGNZFR3KSBBJNEJRFN3MAQL7S46JGFA5I22AUL4OPYL)
- **Deployment transaction:** [View on Stellar Expert](https://stellar.expert/explorer/testnet/tx/1d539b2c8f345601a54e039779a46b4fa14ccfee0e553884c2531bc2caec6f74)

This deploys the contract code only. It does not create or configure a savings group, select a token, enroll members, or make the contract ready to receive funds. Testnet assets have no real-world value.

## Why Soroban for community savings

Rotating savings groups depend on clear rules: who can join, how much members contribute, whose turn comes next, and what happens if a group pauses. Rotula explores encoding a subset of those rules in a Soroban contract so that token movement and group state can be checked on Stellar instead of relying only on an off-chain database.

```text
WhatsApp conversations        Rotula backend               Stellar / Soroban
group coordination    ─────►   member + group records ──► configured token
reminders and help              authorization             contract state/events
                                RPC + confirmation          contributions/payouts
```

The intended savings asset is **USDC on Stellar**, but this contract does not hard-code USDC. Initialization receives a token contract address, and all amounts are integer base units. The calling application must select the correct asset contract and perform exact decimal conversion. Some backend payment and command flows currently use XLM, so the full system's asset configuration is not yet consistent.

## Contract behavior

The `RotulaSavingsContract` supports two group types:

| Group type   | Contribution behavior                                                                                                           | Outgoing funds                                                                                                                                            |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Rotational` | A member must contribute the configured amount once per cycle. The member count is frozen on the first contribution in a cycle. | An administrator-authorized call pays the next member in contract order, using the configured contribution amount multiplied by the frozen member count.  |
| `GoalBased`  | A member-authorized contribution may be any positive amount.                                                                    | A member can withdraw from their recorded savings, subject to available contract tokens and any configured target lock. Rotational payout is unavailable. |

Additional controls include:

- Administrator-authorized initialization and membership changes.
- Member authorization for contributions and withdrawals.
- Expected-recipient checking for rotational payouts.
- Pausing and resuming contract operations, plus a paused-member recovery path for a contribution made in the current cycle.
- Cycle and rotation reset operations controlled by the administrator.
- Events for initialization, membership, contributions, payouts, withdrawals, pause changes, and cycle operations.
- Storage time-to-live extension for group and member state.
- A payout reentrancy guard and checks-effects-interactions ordering around token transfers.

The contract enforces rules on an invocation; it does not run on a wall-clock schedule. `expected_cycle_days` informs the stored cycle-length setting used for TTL calculations, but the contract does not automatically trigger contributions, payouts, or resets. The application coordinates those actions.

## Interface

Primary exported operations:

```text
initialize(
  admin: Address,
  token: Address,
  name: String,
  contribution_amount: i128,
  group_type: GroupType,
  target_amount: Option<i128>,
  lock_until_target: bool,
  expected_cycle_days: Option<u32>
)
add_member(new_member: Address)
remove_member(member_to_remove: Address)
contribute(member: Address, amount: i128)
payout(expected_recipient: Address)
get_next_payout_recipient() -> Address
withdraw_savings(member: Address, amount: i128)
emergency_withdraw(member: Address)
pause()
unpause()
reset_cycle()
reset_rotation()
get_balance() -> i128
get_contribution(member: Address) -> i128
has_received_payout(member: Address) -> bool
```

`GroupType` is `Rotational` or `GoalBased`. Initialization starts with no members; the administrator must add them. State-changing operations require the corresponding Soroban authorization. A successful contract invocation is still only one part of a complete product flow: the application must submit it to the intended network, wait for confirmation, and reconcile the resulting state.

## Rotational cycle notes

The first contribution in a cycle freezes the current member count. Each member can contribute once in that cycle. The administrator then calls `payout(expected_recipient)` in the contract's deterministic member order. The contract computes the payout as:

```text
configured contribution amount × frozen member count
```

The contract checks that the next expected member matches the requested recipient and that the contract's token balance covers that payout. It does not itself schedule payouts or initiate a new rotation. After the payout sequence, the application must coordinate the administrator-authorized `reset_cycle()` and `reset_rotation()` calls at the appropriate time.

Because the contract's pool accounting, contribution readiness, and backend payout lifecycle must work together, treat these rules as code under development. Review and test the full lifecycle before deploying a public instance.

## Prerequisites

| Tool            | Version / Notes                                 | Install                                                                            |
| --------------- | ----------------------------------------------- | ---------------------------------------------------------------------------------- |
| **Rust**        | stable, with the `wasm32v1-none` target         | https://rustup.rs                                                                  |
| **Stellar CLI** | latest, for deploying and invoking the contract | `brew install stellar-cli` or [cargo/docs](https://github.com/stellar/stellar-cli) |
| **Soroban SDK** | 27 (pinned in `contracts/Cargo.toml`)           | pulled by Cargo                                                                    |

Verify your setup:

```bash
rustup target add wasm32v1-none
rustc --version
stellar --version
```

## Build and test

Requires Rust and the Wasm target:

```bash
rustup target add wasm32v1-none
```

From the `contracts/` directory:

```bash
cargo test
cargo build --target wasm32v1-none --release
```

The release Wasm artifact is written to `contracts/target/wasm32v1-none/release/`. This target is required by Soroban SDK 27 with current Rust toolchains; `wasm32-unknown-unknown` is rejected by the SDK on newer Rust versions.

## Integration with Rotula

The [Rotula backend](https://github.com/Rotula-Labs/rotula-api) is responsible for loading the compiled Wasm, deploying contract instances, constructing and simulating Soroban transactions, obtaining authorization, submitting through Soroban RPC, and waiting for confirmation. It also needs to keep contract state and PostgreSQL records reconcilable. The [Rotula frontend](https://github.com/Rotula-Labs/rotula-app) is the web companion; the planned primary member experience is WhatsApp-first.

Integration is still in progress. Before relying on a deployment, align the backend's ABI and membership lifecycle with this interface and verify asset address, issuer, amount precision, transaction authorization, payout readiness, errors, and recovery behavior together.

## Environment Variables

The contract itself reads no environment variables; Soroban contracts receive their configuration through `initialize`, not the host environment. The deployment script and CI read the following:

| Variable                 | Used by                              | Purpose                                                          |
| ------------------------ | ------------------------------------ | ---------------------------------------------------------------- |
| `STELLAR_SOURCE_ACCOUNT` | `contracts/deploy.sh`, CI deploy job | Named Stellar identity (or public key) that signs the deployment |
| `STELLAR_NETWORK`        | `contracts/deploy.sh`                | Target network (`testnet` by default)                            |

Initialize-time arguments (**not** environment variables, but the asset and rules a group is created with) are `admin`, `token`, `name`, `contribution_amount`, `group_type`, `target_amount`, `lock_until_target`, and `expected_cycle_days`. See [Interface](#interface). Signing material lives in Stellar CLI secure storage, never in this repository.

## Security and network use

- Use Stellar Testnet for development. Never include a secret key or phrase in source, logs, issues, or commits.
- Verify the token contract address and asset issuer; an asset code alone does not identify a Stellar asset.
- Confirm all amounts use the configured token's smallest units and stay within integer bounds.
- Review administrator powers, membership changes, payout readiness, pause/recovery behavior, and Soroban storage TTL before any public deployment.
- Contract tests do not establish that the integrated application is safe for production funds.

## Contributing

Open an issue before a larger change. Contract changes should include tests for successful behavior and relevant authorization failures, invalid amounts, membership changes, cycle boundaries, pause/recovery, and token-transfer edge cases. Changes affecting pooled funds or payout order need careful review.

## License

MIT
