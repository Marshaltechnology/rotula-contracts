# Kolo Soroban Contracts — Savings rules on Stellar

> Soroban contract experiments for transparent community savings groups.

Kolo is being built for communities that already save together through Ajo, Esusu, and other rotating savings circles. This repository contains the Rust Soroban contract that models the on-chain side of that idea: a group, its members, a configured Stellar token, contribution state, and the rules governing payouts or goal-based withdrawals.

Stellar is Kolo's intended settlement network. Soroban makes it possible to represent group rules as contract logic and to emit events as state changes. The contract does not replace the WhatsApp experience or Kolo's backend; those systems must still create groups, coordinate members, authorize invocations, submit transactions, and communicate confirmed outcomes.

**Status:** contract code and tests are under active development. This is not a deployed savings product or a production-ready custody system. The backend integration is still being aligned with the current contract interface. Do not use real funds.

## Why Soroban for community savings

Rotating savings groups depend on clear rules: who can join, how much members contribute, whose turn comes next, and what happens if a group pauses. Kolo explores encoding a subset of those rules in a Soroban contract so that token movement and group state can be checked on Stellar instead of relying only on an off-chain database.

```text
WhatsApp conversations        Kolo backend               Stellar / Soroban
group coordination    ─────►   member + group records ──► configured token
reminders and help              authorization             contract state/events
                                RPC + confirmation          contributions/payouts
```

The intended savings asset is **USDC on Stellar**, but this contract does not hard-code USDC. Initialization receives a token contract address, and all amounts are integer base units. The calling application must select the correct asset contract and perform exact decimal conversion. Some backend payment and command flows currently use XLM, so the full system's asset configuration is not yet consistent.

## Contract behavior

The `KoloSavingsContract` supports two group types:

| Group type | Contribution behavior | Outgoing funds |
| --- | --- | --- |
| `Rotational` | A member must contribute the configured amount once per cycle. The member count is frozen on the first contribution in a cycle. | An administrator-authorized call pays the next member in contract order, using the configured contribution amount multiplied by the frozen member count. |
| `GoalBased` | A member-authorized contribution may be any positive amount. | A member can withdraw from their recorded savings, subject to available contract tokens and any configured target lock. Rotational payout is unavailable. |

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

## Build and test

Requires Rust and the Wasm target:

```bash
rustup target add wasm32-unknown-unknown
```

From the `contracts/` directory:

```bash
cargo test
cargo build --target wasm32-unknown-unknown --release
```

The release Wasm artifact is written to `contracts/target/wasm32-unknown-unknown/release/`.

## Integration with Kolo

The [Kolo backend](https://github.com/Stellar-Kolo/kolo-backend) is responsible for loading the compiled Wasm, deploying contract instances, constructing and simulating Soroban transactions, obtaining authorization, submitting through Soroban RPC, and waiting for confirmation. It also needs to keep contract state and PostgreSQL records reconcilable. The [Kolo frontend](https://github.com/Stellar-Kolo/kolo-frontend) is the web companion; the planned primary member experience is WhatsApp-first.

Integration is still in progress. Before relying on a deployment, align the backend's ABI and membership lifecycle with this interface and verify asset address, issuer, amount precision, transaction authorization, payout readiness, errors, and recovery behavior together.

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
