# Gas Specification

This document contains a detailed specification of the way gas is handled within
Sovereign's SDK.

## Definition

Gas is an ubiquitous concept in the blockchain space. It is a measure of the
computational effort required to perform an operation as part of a transaction
execution context. This is used to prevent the network from getting spammed by
regulating the use of computational resources by each participant in the
network.

## High level overview

We have drawn a lot of inspiration from the
[Ethereum gas model](https://ethereum.org/en/developers/docs/gas/) in our gas
mechanism design. Given that Ethereum's gas is well understood and widely used
in the crypto industry, we believe that this will help users onboard more easily
while providing strong security guarantees out-of-the box. We have deliberately
chosen to tweak some concepts that were ill-suited to the rollups built using
Sovereing's SDK. In particular, sorted decreasing order of importance:

- We are using multidimensional gas units and prices.
- We are using a dynamic gas target. Otherwise, the rollups built with
  Sovereign's SDK follow the [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559)
  specification by default.
- Rollup transactions specify a `max_fee`, `max_priority_fee_bips`, and
  _optional gas limit_ `gas_limit`. The semantics of these quantities roughtly
  match their definition in the
  [EIP-1559 specification](https://eips.ethereum.org/EIPS/eip-1559).
- Transaction rewards are decomposed into `base_fee` and `priority_fee`. The
  `base_fee` _is only partially burnt by default_, the remaining amount is used
  _to reward provers/attesters_. The `priority_fee` is used to reward the _block
  sequencers_.
- We are charging gas for every storage access within the module system by
  default.
- Customers of the SDK will have access to wrappers that allow to charge gas for
  hash computation and signature checks.

## A design for multidimensional gas

Sovereign SDK's rollups use multidimensional gas units and prices. For example,
this allows developers to take into account the differences between native and
zero-knowledge computational costs for the same operation. Indeed:

- Hashing is orders of magnitude more expensive when performed inside a
  zero-knowledge circuit. The cost of proving the correct computation of two
  different Hash may also vary much more than the cost of computing the hash
  itself (`Poseidon` or `MiMc` vs `Sha2`).
- Accessing a storage cell for the first time is much more expensive in `zk`
  mode than in `native` mode. But _hot_ storage accesses are practically free in
  zero-knowledge.

**The number and meaning of each dimension is is still not finalized**. The most
recent designs account for 4 dimensions that represent fundamental metrics of
the rollup _which are assumed to vary slowly_ (on a weekly/monthly scale):

- Native computation costs.
- Zk computation costs.
- Storage size (which should be a function of the depth of the Jellyfish Merkle
  tree or NOMT)
- Throughput and DA layer congestion - this dimension should track the
  **long-term network congestion** (at the DA level) and should not be strongly
  affected by local congestion spikes.

We have chosen to follow the
[multi-dimensional EIP-1559](https://ethresear.ch/t/multidimensional-eip-1559/11651)
design for the gas pricing adjustment formulas. In essence:

- We are performing the gas price updates for each dimension separately. In
  other words, each dimension follows a separate uni-dimensional EIP-1559 gas
  price adjustment formula.
- The gas price adjustment formula uses a `gas_target` reference, which is a
  uni-dimensional gas unit that is compared to the gas consumed `gas_used`. The
  `gas_price` is then adjusted to regulate the gas throughtput to get as close
  as possible to the `gas_target`. We have the following invariant:
  `0 <= gas_used_slot <= 2 * gas_target`.
- _Contrarily to Ethereum_, we are planning to design a dynamic `gas_target`.
  The value of the `gas_target` will vary slowly to follow the evolution of the
  rollup metrics we have described above. That way, Sovereign rollups can
  account for major technological improvements in computation (such as zk-proof
  generation throughtput), or storage cost.
- Every transaction has to specify a unidimensional `max_fee` which is the
  maximum amount of _gas tokens_ that can be used to execute a given
  transaction. Similarly, users have to specify a `max_priority_fee_per_gas`
  expressed in basis points which can be used to reward the transaction
  sequencer.
- The final sequencer reward is:
  `seq_reward = min(max_fee - <base_fee, gas_price>, max_priority_fee_per_gas * <base_fee, gas_price>)`.
- Users can provide an optional `gas_limit` field which is a maximum amount of
  gas to be used for the transaction. This quantity is converted to a
  uni-dimensional `remaining_funds` quantity by taking the scalar product with
  the current `gas_price`.
- If users provide the `gas_limit`, the rollup checks that
  `<gas_limit, current_gas_price> <= max_fee` (ie, the scalar product with the
  current `gas_price`). If the check fails, the associated transaction is not
  executed and the rollup raises a
  `ReserveGasErrorReason::CurrentGasPriceTooHigh` error.

## Charging gas for state accesses.

State accessors such as the `WorkingSet` or the `PreExecWorkingSet` charge some
gas whenever state is modified. If these accessors run out of gas, they return a
`StateAccessorError` and the execution gets reverted (or the sequencer is
penalized). Some state accessors - like `StateCheckpoint`, the `TxScratchpad` or
the `ApiStateAccessor` - don't charge for gas for state accesses. In that case,
the access methods return a `Result<T, Infallible>` type which can be unwrapped
safely using `unwrap_infallible`.

For now, we are enforcing simple cached access patterns - we are refunding some
gas if the value that is accessed/modified is _hot_ (ie has been already
accessed and is cached).

## Gas rewards.

The gas consumed during transaction execution is used to reward both
provers/attesters and block sequencers. The `base_fee`, ie the total amount of
gas consumed by the transaction execution is partially burnt (the amount to burn
is specified by the `PERCENT_BASE_FEE_TO_BURN` constant), and the remaining
portion is locked in a reward pool to be redeemed by provers/attesters. The
`priority_fee` is also partially burnt and used to reward block sequencers.

## Additional data structures that can be used to charge gas.

We have a couple of additional data structures that can be used to charge gas.
These are:

- `MeteredHasher`: a wrapper structure that can be used to charge gas for hash
  computation.
- `MeteredSignature`: a wrapper structure that can be used to charge gas for
  signature checks.
- `MeteredBorshDeserialize`: a supertrait that can be used to charge gas for
  structures implementing `BorshDeserialize`.

## Structure of the implementation

The core of the gas implementation is located within the `sov-modules-api` crate
in the following modules/files:

- `module-system/sov-modules-api/src/common/gas.rs`: contains the implementation
  of the `Gas` and `GasMeter` traits. These are the core interfaces that are
  consumed by the API. The `Gas` trait defines the way users can interact with
  multidimensional gas units. The `GasMeter` is the interface implemented by
  every data structure that contains or consumes gas (such as the `WorkingSet`
  which contains a `TxGasMeter`, or the `PreExecWorkingSet` that may contain a
  `SequencerStakeMeter`).
- `module-system/sov-modules-api/src/common/hash.rs`: contains the
  implementation of the `MeteredHasher` which is a wrapper structure that can be
  used to charge gas for hash computation.
- `module-system/sov-modules-api/src/transaction.rs`: contains the
  representation of the transaction type that is used within the SDK. These
  structures contain the `max_fee`, `max_priority_fee_bips` and `gas_limit`
  fields that represent the maximum amount of gas tokens to use for the
  transaction, the maximum priority fee to pay the sequencer (in basis points),
  and an optionnal multidimensional gas limit (ie the maximum amount of gas to
  be consumed for this transaction).

Outside of the `sov-modules-api`, within the module system:

- `module-system/module-implementations/sov-chain-state/src/gas.rs`:
  `compute_base_fee_per_gas` contains the implementation of the gas price update
  which follows our modified version of the `EIP-1559`. The gas price is updated
  within the `ChainState`'s module lifecycle hooks
  (`ChainState::begin_slot_hook` updates the gas price,
  `ChainState::end_slot_hook` updates the gas consumed by the transaction).
- `module-system/module-implementations/sov-sequencer-registry/src/capabilities.rs`:
  contains the implementationn of the `SequencerStakeMeter` which is the data
  structure used to meter the sequencer stake before the transaction's execution
  starts.
