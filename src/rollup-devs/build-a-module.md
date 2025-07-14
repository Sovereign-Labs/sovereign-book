# Implementing a Module

A module is the basic unit of functionality in the Sovereign SDK. A module is a
collection of values stored in state, and an optional `call` function which
allows users to interact with that state onchain. Modules also provide custom
API support and built-in indexing.

In this section, we'll describe how to implement a module and take full
advantage of the Sovereign SDKs builtin functionality.

## The Module Struct

A typical module definition looks like this.

```rust
#[derive(Clone, ModuleInfo, ModuleRestApi)]
pub struct Example<S: Spec> {
    /// The ID of the module.
    #[id]
    pub(crate) id: ModuleId,

    /// A reference to the Bank module, so that we can transfer tokens.
    #[module]
    pub(crate) bank: sov_bank::Bank<S>,

	/// A mapping from addresses to users
    #[state]
    pub(crate) users: StateMap<S::Address, User<S>>,
}
```

There are a few things to notice in this snippet.

### Derived Traits

First, we derive the `ModuleInfo` and `ModuleRestApi` traits.

You should always derive `ModuleInfo` on your module, since it does important
work like laying out your state values in the database. If you forget to derive
this trait, the SDK will throw a helpful error.

The `ModuleRestApi` trait is optional but very useful. It tries to generate
RESTful API endpoints for all of the `#[state]` items in your module. Each
item's endpoint will have the name
`{hostname}/modules/{module-name}/{field-name}/`, with all items automatically
converted to `kebab-casing`. For example, you could query an item from the
`known_sequencers` state value shown in the snippet above at the path
`/modules/sequencer-registry/known-sequencers/{address}`.

Note that `ModuleRestApi` can't always generate endpoints for you. If it can't
figure out how to generate an endpoint for a particular state value, it will
simply skip it by default. (Don't worry, you can always add endpoints manually
if you need to! We'll cover this in the next section.). If you want to override
the default behavior and throw a compiler error if endpoint generation fails,
you can add the `#[rest_api(include)]` attribute on your state value like this:

```rust
#[derive(Clone, ModuleInfo, ModuleRestApi)]
pub struct Example<S: Spec> {
    // Redundant code elided here...
	#[rest_api(include)]
    #[state]
    pub(crate) users: StateMap<S::Address, User<S>>,
}
```

See the
[documentation](https://github.com/Sovereign-Labs/sovereign-sdk-wip/blob/1210acb5f5d8ca408ebaea40db3a75e5d2521491/crates/module-system/sov-modules-api/src/reexport_macros.rs#L387)
on `ModuleRestApi` for more details.

### The `Spec` Generic

Next, notice that the module is parameterized by `Spec` type. This paramter
gives you access to a bunch of handy types and cryptographic traits that are
used on the rollup. These includes the `Address` type used on the rollup, as
well as information about the underlying DA layer, like its `Da::Address` type,
`BlockHeader` type, and more. By being generic over the `Spec`, you ensure that
you can easily change things like the `Address` format used by your module later
on.

See the
[`Spec` trait docs](https://github.com/Sovereign-Labs/sovereign-sdk-wip/blob/1210acb5f5d8ca408ebaea40db3a75e5d2521491/crates/module-system/sov-modules-api/src/module/spec.rs#L28-L29)
for more details.

### `#[state]`, `#[module]`, and #[id] fields

Modules may have as many `#[state]` and `#[module]` fields as they wish, but
they must have exactly one `#[id]`. The `ModuleInfo` macro will automatically
generate a unique identifier for the module and store it in the `id` field.

`#[module]` fields are very straightforward. Adding a dependency on another
module will allow your module to access any of it's public fields and/or public
methods. The only downside is that any users of your module will also have to
include the dependency in their rollup.

`#[state]` fields are the most interesting. They define the layout of the state
that your module will store. There are three primary kinds of `#[state]`:
`StateValue`s which store a single item, and `StateMap`s which store a mapping
from keys to values, and `StateVec`s which store a series of elements, keyed by
index.

Each kind of state item accepts an optional `Codec` parameter, which determines
how the state value is encoded and decoded for storage. By default, we use
`Borsh` for everything, but the SDK also provides a `BcsCodec` which is
compatible with the widely used `serde` library. If you want to store a type
from a 3rd-party library in you module, you might need to specify the BcsCodec
like this:

```rust
use some_third_party_crate::SomeType;
#[derive(Clone, ModuleInfo, ModuleRestApi)]
pub struct MyModule<S: Spec> {
    // Redundant code elided here...
    #[state]
    pub(crate) my_map: StateValue<SomeType, BcsCoded>,
}
```

Note that this is only necessary if the compiler warns you that the type doesn't
suppor `BorshSerialize` or `BorshDeserialize`. If the compiler is happy, you
don't need to override the default.

The last important thing to note about state is that for each kind of item
(`Value/Map/Vec`), we provide a corresponding `AccessoryState{Item}` type.
Accessory state looks exactly like regular state, but it's _write only_ during
transaction execution. Items stored in accessory state can be queried via the
API, but they don't have to be replicated by everybody running your rollup since
transactions can't read their contents. This makes accessory state much more
efficient than normal state for indexing and other off-chain tasks.

```rust
use some_third_party_crate::SomeType;
#[derive(Clone, ModuleInfo, ModuleRestApi)]
pub struct MyModule<S: Spec> {
    // Redundant code elided here...
	/// Stores a record of the purchase history of users that can be queried from the API
	/// but doesn't exist on-chain. This is much cheaper than maintaining the map on chain.
    #[state]
    pub(crate) token_buyers: AccessoryStateMap<S::Address, Vec<Price>>,
}
```

## Module Trait Implementation

Once you've defined your module's state layout, you need a way for users to
interact with it.

That's where the `Module` trait comes in. This trait has two important methods,
`genesis` and `call`. You can see a simplified version of the trait below:

```rust
trait Module {
    /// The configuration needed to initialize the module.
    type Config;

    /// A module defined argument to the call method. This is usually an enum,
    /// with  one variant for each action a user might take.
    type CallMessage: Debug + BorshSerialize + BorshDeserialize + Clone;

    /// A module defined event resulting from a call method. Events invisible to on-chain
    /// logic, but they're very useful for indexing.
    type Event: Debug + BorshSerialize + BorshDeserialize + 'static + core::marker::Send;

    /// Genesis is called once when a rollup is deployed.
    ///
    /// You should use this function to initialize all of your module's `StateValue`s and run any other
    /// one-time setup. Since this function runs only once, it's perfectly acceptible to do expensive operations
    /// here. Note that your function should still be deterministic, however.
    fn genesis(
        &self,
        genesis_block_header: &<<Self::Spec as Spec>::Da as DaSpec>::BlockHeader,
        config: &Self::Config,
        state: &mut impl GenesisState<Self::Spec>,
    ) -> Result<(), ModuleError>;


    /// Call accepts a `CallMessage` and executes it, changing the state of the module and emitting events.
    fn  call(&self,
        message: Self::CallMessage,
        context: &Context<Self::Spec>,
        state: &mut impl TxState<Self::Spec>,
    ) -> Result<(), ModuleError>;
}
```

### Genesis

As you might expect, the `genesis` function is called exactly once when the
rollup is initialized. It lets the person deploying your module set any initial
configuration and is responsible for making sure that any state values in your
module are initialized (if necessary). Since `genesis` is only called once, you
don't need to worry about efficiency. However, you do need to be careful not to
do anything non-determinstic. That means no network requests, and no use of
randomness!

As you can see, the `genesis` function gets passed a user-defined `Config`
value. As a module author, you can put anything you want here. Deployers of your
rollup will need to instantiate the config at genesis, and you can use it to do
initialization. If your module doesn't need configuring, you can juse use the
empty type `()`.

`genesis` also accepts an argument which implements the `GenesisState` trait.
This argument allows you to read and write to state values in your module and
its dependencies. A typical genesis config definition and genesis function
looklike this:

```rust
// Adapter from the `SequencerRegistry` module
pub struct SequencerConfig<S: Spec> {
    /// The rollup address of the sequencer.
    pub seq_rollup_address: S::Address,
    /// The Data Availability (DA) address of the sequencer.
    pub seq_da_address: <S::Da as DaSpec>::Address,
    /// Initial sequencer bond
    pub seq_bond: u128,
}

pub(crate) fn genesis(
    &self,
    _genesis_rollup_header: &<<S as Spec>::Da as DaSpec>::BlockHeader,
    config: &SequencerConfig,
    state: &mut impl GenesisState<S>,
) -> Result<()> {
    self.register_staker(
        &config.seq_da_address,
        Amount::new(config.seq_bond),
        config.seq_rollup_address.clone(),
        state,
    )?;
}
```

### Call

The `call` function provides the transaction processing logic for your module.
It accepts a structured input from a user and a `Context` which contains
metadata including the sender address. In response to a `call`, modules may
update their state as well as emitting `Events` - structured key-value pairs
which are returned to the user and can be queried over the REST API.

If your call function returns an error, all of its state changes are
automatically reverted and any events are discarded. However, any logs generated
by the transaction will still be visible to the node operator. (More on
`logging` later.)

You can define the `CallMessage` accepted by your module to be any type you
wish, but an enum is usually best. Be sure to implement `borsh` and `serde`
serialization for your type, as well as `schemars::JsonSchema` and
`sov_modules_macros::UniversalWallet`. This will ensure that it's maximally
portable across languages and frontends, making it easy for users to securely
generate, review, sign, and send transactions to your module.

The `Bank` module provides a very typical example of a `call` implementation:

```rust
fn call(&self, msg: CallMessage, context: &Context<Self::Spec>, state: &mut impl TxState<S>)
-> Result<(), Error> {
    match msg {
        call::CallMessage::Transfer { to, coins } => { Ok(self.do_transfer(context.sender(), &to, &coins.token_id, coins.amount, state)?) }
        // Other variants omitted for brevity
    }
}

fn do_transfer(&self, from: S::Address, to: S::Address, token_id: &TokenId, amount: Amount, state: &mut impl StateAccessor) -> anyhow::Result<()> {
    if from == to || amount == 0{
        return Ok(());
    }
    // Use a helper to compute the new `from` balance, throwing an error on insufficient funds
    let new_from_balance = self.decrease_balance_checked(token_id, from, amount, state)?;

    // Get the current `to` balance and compute the new one
    // Note that the `get` function can return an error if the call runs out of gas
    let current_to_balance = self
        .balances
        .get(&(to, token_id), state)?
        .unwrap_or(Amount::ZERO);
    let to_balance = current_to_balance.checked_add(amount).with_context(|| {
        format!(
            "Account balance overflow for {} when adding {} to current balance {}",
            to, amount, current_to_balance
        )
    })?;

    // Update both balances and emit an event, reverting on error.
    self.balances.set(&(from, token_id), &new_from_balance, state)?;
    self.balances.set(&(to, token_id), &to_balance, state)?;
    self.emit_event(
        state,
        Event::TokenTransferred {
            from: sender.as_token_holder().into(),
            to: to.into(),
            coins,
        },
    );
    Ok(())
}

```

Just like Ethereum `smart contracts` and Solana `programs`, modules accept
inputs that are pre-validated by the chain. That means your module does _not_
need to worry about authenticating the transaction. In most cases, you also
don't need to worry about manually metering resource consumption. The SDK will
automatically charge gas for any state accesses and deduct the cost from the
sender's balance. However, if your module does any very heavy computation you
may need to meter that explicitly using the `Module::charge_gas` function.

## Optional Functionality - Hooks

In addition to `call` modules may _optionally_ implement `Hooks`. Hooks run at
the begining and end of every rollup block and every transaction. `BlockHooks`
are great for taking actions that need to happen before or after any
transactions execute in a block - but be careful, no one pays for the
computation done by `BlockHooks`, so doing any heavy computation can make your
rollup vulnerable to DOS attacks.

`TxHooks` are useful for checking, or to allow your module to monitor actions
being taken by other modules. Unlike `BlockHooks`, `TxHooks` are paid for by the
user who sent each transaction.

The `FinalizeHook` is great for doing indexing. It can only modify
`AccessoryState`, which makes it cheap to run but means that the results will
only be visible via the API

Using the hooks is somewhat unusual - most applications only need to modify
their state in response to user actions - but it's a powerful tool in some
cases. See the documentation on
[`BlockHooks`](https://github.com/Sovereign-Labs/sovereign-sdk-wip/blob/1210acb5f5d8ca408ebaea40db3a75e5d2521491/crates/module-system/sov-modules-api/src/hooks.rs#L76)
and
[`TxHooks`](https://github.com/Sovereign-Labs/sovereign-sdk-wip/blob/1210acb5f5d8ca408ebaea40db3a75e5d2521491/crates/module-system/sov-modules-api/src/hooks.rs#L12)
and
[`FinalizeHook](https://github.com/Sovereign-Labs/sovereign-sdk-wip/blob/1210acb5f5d8ca408ebaea40db3a75e5d2521491/crates/module-system/sov-modules-api/src/hooks.rs#L120)
more details.

## Advanced Functionalty - native only code

In this section, we'll describe more advanced functionality - adding custom
APIs, supporting JSON-RPC, and instrumenting your code for debugging and
optimization.

Any code that you write following the guide in this section needs to be gated
behind the `#[cfg(feature = "native")]` flag, which signals to the SDK that the
code is not part of the module's state transition function and is not relevant
to any questions about the rollup's current state. This means that it will be
excluded from `zk-proof` generation (if the rollup is a `zk-rollup`) or
challenges (if the rollup is optimistic).

### Adding Custom REST APIs

You can easily add custom APIs to your module by implementing the
`HasCustomRestApi` trait. This trait has two methods - one which actually
implements the routes, and an optional one which provides an `OpenApi` spec. YOu
can see a good example in the `Bank` module:

```
impl<S: Spec> HasCustomRestApi for Bank<S> {
    type Spec = S;

    fn custom_rest_api(&self, state: ApiState<S>) -> axum::Router<()> {
        axum::Router::new()
            .route(
                "/tokens/:tokenId/total-supply",
                get(Self::route_total_supply),
            )
            .with_state(state.with(self.clone()))
    }

    fn custom_openapi_spec(&self) -> Option<OpenApi> {
        let mut open_api: OpenApi =
            serde_yaml::from_str(include_str!("../openapi-v3.yaml")).expect("Invalid OpenAPI spec");
        for path_item in open_api.paths.paths.values_mut() {
            path_item.extensions = None;
        }
        Some(open_api)
    }
}

async fn route_balance(
    state: ApiState<S, Self>,
    mut accessor: ApiStateAccessor<S>,
    Path((token_id, user_address)): Path<(TokenId, S::Address)>,
) -> ApiResult<Coins> {
    let amount = state
        .get_balance_of(&user_address, token_id, &mut accessor)
        .unwrap_infallible() // State access can't fail because no one has to pay for gas.
        .ok_or_else(|| errors::not_found_404("Balance", user_address))?;

    Ok(Coins { amount, token_id }.into())
}
```

REST API methods get access to an `ApiStateAccessor`. This special struct gives
you access to both normal and `Accessory` state values. You can freely read and
write to state during your API calls, which makes it easy to reuse code from the
rest of your module. However, it's important to remember API calls do _not_
durably mutate state. Any state changes are thrown away at the end of the
request.

If you implement a custom REST API, your new routes will be automatically nested
under your module's router. So, in the following example, the
`tokens/:tokenId/total-supply` function can be found at
`/modules/bank/tokens/:tokenId/total-supply`. Similarly, your OpenApi spec will
get combined with the auto-generated one automatically.

Note that for for custom REST APIs, you'll need to manually write an `OpenApi`
specification if you want client support.

### Legacy RPC Support

In addition to custom RESTful APIs, the Sovereign SDK lets you create JSON-RPC
methods. This is useful to provide API compatibility with existing chains like
Ethereum and Solana, but we recommend using REST APIs whenever compatibility
isn't a concern.

To implement RPC methods, simply annotate an `impl` block on your module with
the `#[rpc_gen(client, server)]` macro, and then write methods which accept an
`ApiStateAcessor` as their final argument and return an `RpcResult`. You can see
some examples in the `Evm` module.

```rust
#[rpc_gen(client, server)]
impl<S: Spec> Evm<S> {
    /// Handler for `net_version`
    #[rpc_method(name = "eth_getStorageAt")]
    pub fn get_storage_at(
        &self,
        address: Address,
        index: U256,
        state: &mut ApiStateAccessor<S>,
    ) -> RpcResult<U256> {
        let storage_slot = self
            .account_storage
            .get(&(&address, &index), state)
            .unwrap_infallible()
            .unwrap_or_default();
        Ok(storage_slot)
    }
}
```

## Operationalizing

In this section, we'll describe how to reason about and measure your module's
performance.

### Understanding Performance

**State Access**

The vast majority of the cost of executing a Sovereign SDK transaction comes
from state accesses. When call `item.set(&value)`, the SDK serializes your value
and stores the bytes in cache. When time you access a value using `item.get()`,
the SDK deserializes a fresch copy of your value from the bytes held in cache,
falling back to disk if necessary.

Each time you access a value that's not in cache, the SDK has to generate a
merkle proof of the value, which it will consume when it's time to generate a
zero-knowledge proof. Similarly, each time you write a new value, the SDK has to
generate a merkle update proof. This makes reading/writing to a `hot` value, at
least an order of magnitude cheaper than writing to a `cold` one (where `hot`
means that the value has already been accessed in the current block.) So, if you
have state items that are frequently accessed together, it's a good idea to
bundle them into a single `StateValue` or store them under the same key in a
`StateMap`.

As a rule of thumb, for each 10% locality, you should be willing to add an extra
200 bytes to your `StateValue`. In other words, if two values are accessed
together 30% of the time, you should put them together unless either of the
state items is bigger than 600 bytes. (Exception: If two items are always
accessed together, you should always group them together - no questions asked).

**Cryptography**

The other common source of performance woes is heavy-duty cryptography. If you
need to do any cryptographic operations, check whether the `Spec` trait provides
a method in its `Spec::CryptoSpec` that already does what you want. If it does,
use that - the SDK will ensure you get an implementation which is optimized for
the SDK's peculiar requirements. If you need access to more exotic cryptography,
you can use pretty much any existing Rust library - but be aware that the
performance penalty might be severe when it comes time to prove your module's
execution, which could limit your total throughput. If you do need advanced
cryptography, you may need to pick an implementation that's suited to a
particular `ZKVM` (like `SP1` or `Risc0`) and only use that vm with your module.

### Instrumenting Your Module

TODO

- Metrics
- Logging/traces

```

```
