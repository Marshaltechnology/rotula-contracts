# Kolo Soroban Savings Contracts

**Kolo is building community savings on Stellar.** This repository contains the Rust Soroban contract for savings groups. Stellar provides the asset and settlement layer; Soroban holds group state and enforces the rules for contributions, ordered payouts, withdrawals, and cycle changes.

Kolo's product is designed for communities that already save together through Ajo/Esusu-style circles. A group agrees on a contribution amount and membership, members contribute the selected Stellar asset, and the contract enforces a deterministic payout rotation. Goal-based groups provide a separate savings mode with target and withdrawal controls.

## Kolo repositories

- [Frontend](https://github.com/Stellar-Kolo/kolo-frontend) — member and admin web experience.
- [Backend](https://github.com/Stellar-Kolo/kolo-backend) — WhatsApp, Stellar, and Soroban orchestration.
- [Soroban contracts](https://github.com/Stellar-Kolo/kolo-contracts) — this Rust contract project.

## Contract capabilities

- Initialize a group with an administrator, Stellar token contract, name, contribution amount, and group configuration.
- Add and remove members with administrator authorization.
- Accept member-authorized contributions and transfer the configured token into the contract.
- In rotational groups, require the exact configured contribution amount and reject a second contribution from a member during the same cycle.
- Pay the next member in contract member order. The administrator authorizes the payout, and the contract checks the expected recipient and available pool balance.
- Withdraw savings in GoalBased groups, with an optional target lock. Rotational groups cannot use this withdrawal method.
- Pause the contract, allow a member to recover their current-cycle contribution while paused, and resume operations through administrator authorization.
- Emit events for initialization, membership changes, contributions, payouts, withdrawals, and cycle operations.

## Stellar asset units

The contract accepts amounts as `i128` integers in the token's smallest units. It does not assume a particular asset or decimal count: the token contract is supplied during initialization. The calling application must choose the intended Stellar asset and convert display amounts to that asset's base units consistently. Kolo's product brief targets USDC, while some current backend flows still refer to XLM; confirm the configured asset and conversion rules before deployment.

## Contract interface

The primary exported operations are:

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

`GroupType` is `Rotational` or `GoalBased`. Initialization and state-changing methods require the appropriate Soroban authorization. The group administrator must explicitly add members; initialization does not enroll them automatically.

For a rotational group, the contract freezes the member count when the first contribution of a cycle arrives. A payout transfers `contribution_amount × frozen_member_count` to the next member in the contract's member list. After all recipients have been paid, the application coordinates `reset_cycle()` and `reset_rotation()` to start a new contribution cycle and rotation. These calls are administrator-authorized; the contract does not schedule them by itself.

## Build and test

Requires Rust and the Wasm target:

```bash
rustup target add wasm32-unknown-unknown
```

From `contracts/`:

```bash
cargo test
cargo build --target wasm32-unknown-unknown --release
```

The release Wasm artifact is written under `contracts/target/wasm32-unknown-unknown/release/`.

## Deployment and integration

The application is responsible for deploying the Wasm, creating a contract instance, initializing it with the exact ABI above, adding authorized members, and submitting signed invocations through a Soroban RPC endpoint. The backend integration is still being aligned with this contract interface; a successful local contract test does not by itself mean the complete Kolo savings flow is deployed or production-ready.

Use Stellar Testnet for development. Review token address, decimal conversion, authorization, storage TTL, payout order, and emergency behavior before any public-network deployment. Never commit secret keys or use production credentials in tests.

## Contributing

Open an issue before larger changes. Contract changes should include Soroban tests for success, authorization failures, invalid amounts, cycle boundaries, and token-transfer edge cases. Changes that affect pooled funds or payout ordering need careful review.

## License

MIT
