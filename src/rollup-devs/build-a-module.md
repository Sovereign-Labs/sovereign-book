# Implementing a Module

A module is the basic unit of functionality in the Sovereign SDK. A module
defines some values to store in state, and optionally references some other
modules.

## The Module Struct

A typical module definition looks like this.

```rust
/// The `sov-sequencer-registry` tracks which addresses can sequence transactions.
#[derive(Clone, ModuleInfo, ModuleRestApi)]
pub struct SequencerRegistry<S: Spec> {
    /// The ID of the `sov_sequencer_registry` module.
    #[id]
    pub(crate) id: ModuleId,

    /// A reference to the Bank module, so that we can transfer tokens.
    #[module]
    pub(crate) bank: sov_bank::Bank<S>,

	/// A list of known sequencers.
    #[state]
    pub(crate) known_sequencers: StateMap<S::Address, KnownSequencer<S>>,
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
pub struct SequencerRegistry<S: Spec> {
    // Redundant code elided here...
	#[rest_api(include)]
    #[state]
    pub(crate) known_sequencers: StateMap<<S::Da as DaSpec>::Address, KnownSequencer<S>>,
}
```

See the
[documentation](https://github.com/Sovereign-Labs/sovereign-sdk-wip/blob/1210acb5f5d8ca408ebaea40db3a75e5d2521491/crates/module-system/sov-modules-api/src/reexport_macros.rs#L387)
on `ModuleRestApi` for more details.

### The `Spec` Generic

Next, notice that the module is parameterized by `Spec` type. This paramter
gives you access to a bunch of handy types and cryptographic traits that are
used on the rollup. By being generic over the `Spec`, you ensure that you can
easily change things like the `Address` format used by your module later on.

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
`genesis` and `call`. You can see a (very slightly) simplified version of the
trait below:

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


    /// `call` accepts a `CallMessage` and executes it, changing the state of the module and emitting events.
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

CallMessage type definition - including JsonSchema, Borsh, and FromStr
requirements plus the UniversalWallet State management and gas consumption
Emitting events

## Optional Functionality - Hooks

## Advanced Functionalty - native only code

### Adding Custom REST APIs

### Legacy RPC Support

## Operationalizing

### Understanding Performance

- State is flushed to disk each block
- On the first read, state is fetched from disk/merkle
- Subsequent reads/writes hit cache, still pay for serde.
- Transactions execute sequentially

### Instrumenting Your Module

TODO

- Metrics
- Logging/traces
