## Gas overview and blessed values

Sovereign's SDK transactions should specify gas parameters in a similar way to
Ethereum. When submitting a transaction, you need to specify a handful of gas
parameters (that are stored in a structure called `TxDetails`) that depend on
the rollup settings but also on the type of call message to execute. We also
have to make sure that the sender holds enough gas tokens in its bank balance to
make sure that the transaction is not rejected due to insufficient funds.
Finally, sequencers need to stake enough tokens to pay for the transaction
pre-execution checks (like signature verification, deserialization, etc.).

This can be quite overwhelming at first glance, hence we provide here a quick
summary of the gas parameters with their respective blessed values (this should
be enough to execute most transactions that are not compute/storage intensive),

First, let's look at the gas parameters that are required to submit a
transaction (in the `TxDetails` structure):

```rust
use sov_modules_api::Spec;
use sov_modules_api::transaction::PriorityFeeBips;

/// Contains details related to fees and gas handling.
pub struct TxDetails<S: Spec> {
    /// The maximum priority fee that can be paid for this transaction expressed as a basis point percentage of the gas consumed by the transaction.
    /// Ie if the transaction has consumed `100` gas tokens, and the priority fee is set to `100_000` (10%), the
    /// gas tip will be `10` tokens.
    pub max_priority_fee_bips: PriorityFeeBips,
    /// The maximum fee that can be paid for this transaction expressed as a the gas token amount
    pub max_fee: u64,
    /// The gas limit of the transaction.
    /// This is an optional field that can be used to provide a limit of the gas usage of the transaction
    /// across the different gas dimensions. If provided, this quantity will be used along
    /// with the current gas price (`gas_limit *_scalar gas_price`) to compute the transaction fee and compare it to the `max_fee`.
    /// If the scalar product of the gas limit and the gas price is greater than the `max_fee`, the transaction will be rejected.
    /// Then up to `gas_limit *_scalar gas_price` gas tokens can be spent on gas execution in the transaction execution - if the
    /// transaction spends more than that amount, it will run out of gas and be reverted.
    pub gas_limit: Option<S::Gas>,
    /// The ID of the target chain.
    pub chain_id: u64,
}
```

- The `max_fee` parameter is the maximum amount of gas expressed in gas tokens
  that can be charged for the transaction execution.
- The `max_priority_fee` parameter is the maximum percentage (expressed in basis
  points) of the total gas consumed by the transaction execution that should be
  paid to reward the sequencer. This parameter can have any value because there
  is a safety mechanism that prevents the user from paying more than the
  `max_fee` in total.
- The `gas_limit` parameter is the maximum amount of gas (expressed in
  multidimensional gas units) that can be consumed by the transaction execution.
  This parameter is optional and can be left unspecified. In the future, we will
  add support for automatically computing this parameter from transaction
  simulation.
- The `user_balance` parameter is the balance of the sender's account (for the
  gas token) in the rollup's bank.
- The `sequencer_balance` parameter is the balance of the sequencer's account
  (for the gas token) in the rollup's bank.
- The `sequencer_stake` parameter is the staked amount of the sequencer in the
  `sequencer_registry` module.

_Blessed gas parameters:_

| Parameter         | Value                               |
| ----------------- | ----------------------------------- |
| max_fee           | 100_000_000                         |
| max_priority_fee  | any (50_000 is a reasonable choice) |
| gas_limit         | None                                |
| user_balance      | 1_000_000_000                       |
| sequencer_balance | 1_000_000_000                       |
| sequencer_stake   | 100_000_000                         |

Note also that:

- The `base_fee_per_gas` parameter (whose initial value `INITIAL_GAS_LIMIT` is
  set by the rollup in the `constants.toml`) roughtly corresponds to the
  rollup's gas price and is an internal parameter of the rollup.
- A batch can consume up to `INITIAL_GAS_LIMIT` gas units of gas, and the gas
  target is `1/ELASTICITY_MULTIPLIER` times that value (for each dimension).
- The `base_fee_per_gas` is dynamically adjusted based on the gas consumption of
  the batch. The adjustment follows the EIP-1559 which makes it goes down if the
  batch consumes more gas than the target (and respectively up if the batch
  consumes less gas than the target).

The [gas specification](./sdk-contributors/gas.md) provides a detailed
description of the gas mechanisms used within the SDK.
