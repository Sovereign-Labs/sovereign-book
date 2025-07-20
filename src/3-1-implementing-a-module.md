# Implementing a Module

A module is the basic unit of functionality in the Sovereign SDK. It's a self-contained piece of onchain logic that manages its own state and defines how users can interact with it.

The best way to learn how modules work is to build one. In this section, we'll create a simple but complete `ValueSetter` module from scratch. This module will allow a designated admin address to set a `u32` value in the rollup's state. After the walkthrough, we'll dive deeper into each of the concepts introduced. 

#### Want to Code Along?

For those who learn best by doing, we recommend using our starter template. It has all the boilerplate and dependencies pre-configured, so you can get straight to the code.
Simply clone the `sov-rollup-starter` repository and navigate to the `value-setter` example module. All the code in this guide will be written within this directory.

```bash
# 1. Clone the starter repo
git clone https://github.com/Sovereign-Labs/sov-rollup-starter.git

# 2. Navigate to the example module
cd sov-rollup-starter/examples/value-setter/
```

## A Step-by-Step Walkthrough: The `ValueSetter` Module

### 1. Define the Module Struct

First, we define the module's structure and the state it will manage. This struct tells the SDK what data to store onchain.

```rust
use sov_modules_api::{Module, ModuleId, ModuleInfo, StateValue, Spec};

// This is the struct that will represent our module.
// It must derive `ModuleInfo` to be a valid module.
#[derive(Clone, ModuleInfo)]
pub struct ValueSetter<S: Spec> {
    /// The `#[id]` attribute is required and uniquely identifies the module instance.
    #[id]
    pub id: ModuleId,

    /// The `#[state]` attribute marks a field as a state variable.
    /// `StateValue` stores a single, typed value.
    #[state]
    pub value: StateValue<u32>,

    /// We'll also store the address of the admin who is allowed to change the value.
    /// S:Address is the address type of our rollup. More on `Spec` later.
    #[state]
    pub admin: StateValue<S::Address>,
}
```

### 2. Define Types for the `Module` Trait

Next, we define the associated types required by the `Module` trait: its configuration, its callable methods, and its events.

```rust
// Continuing in the same file...
use schemars::JsonSchema;
use sov_modules_api::macros::{serialize, UniversalWallet};

// The configuration for our module at genesis. This will be deserialized from `genesis.json`.
#[derive(Clone, Debug, PartialEq, Eq)]
#[serialize(Borsh, Serde)]
#[serde(rename_all = "snake_case")]
pub struct ValueSetterConfig<S: Spec> {
    pub initial_value: u32,
    pub admin: S::Address,
}

// The actions a user can take. Our module only supports one action: setting the value.
#[derive(Clone, Debug, PartialEq, Eq, JsonSchema, UniversalWallet)]
#[serialize(Borsh, Serde)]
#[serde(rename_all = "snake_case")]
pub enum CallMessage {
    SetValue(u32),
}

```

### 3. Implement the `Module` Trait Logic

With our types defined, we can now implement the `Module` trait itself.

```rust
use anyhow::Result;
use sov_modules_api::{Context, GenesisState, TxState};

// Now, we implement the `Module` trait.
impl<S: Spec> Module for ValueSetter<S> {
    type Spec = S;
    type Config = ValueSetterConfig<S>;
    type CallMessage = CallMessage;
    // We'll add events later...
    type Event = ();

    // `genesis` is called once when the rollup is deployed to initialize the state.
    fn genesis(&mut self, _header: &<S::Da as sov_modules_api::DaSpec>::BlockHeader, config: &Self::Config, state: &mut impl GenesisState<S>) -> Result<()> {
        self.value.set(&config.initial_value, state)?;
        self.admin.set(&config.admin, state)?;
        Ok(())
    }

    // `call` is called when a user submits a transaction to the module.
    fn call(&mut self, msg: Self::CallMessage, context: &Context<S>, state: &mut impl TxState<S>) -> Result<()> {
        match msg {
            CallMessage::SetValue(new_value) => {
                self.set_value(new_value, context, state)?;

                Ok(())
            }
        }
    }
}
```

### 4. Write the Business Logic

The final piece is to write the private `set_value` method containing our business logic.

```rust
impl<S: Spec> ValueSetter<S> {
    fn set_value(&mut self, new_value: u32, context: &Context<S>, state: &mut impl TxState<S>) -> Result<()> {
        // `get_or_err` returns a nested Result, so we use `??` to unpack it.
        let admin = self.admin.get_or_err(state)??;

        // Check if the sender is admin
        if admin != *context.sender() {
            return Err(anyhow::anyhow!("Only the admin can set the value.").into());
        }

        // Update the `value` in our state
        self.value.set(&new_value, state)?;

        Ok(())
    }
}
```

With that, you've implemented a complete module! Now, let's break down the concepts we used in more detail.

---

## Anatomy of a Module: A Deeper Look

### Derived Traits: `ModuleInfo` and `ModuleRestApi`

You should always derive `ModuleInfo` on your module, since it does important
work like laying out your state values in the database. If you forget to derive
this trait, the SDK will throw a helpful error.

The `ModuleRestApi` trait is optional but highly recommended. It automatically generates RESTful API endpoints for the `#[state]` items in your module. Each item's endpoint will have the name `{hostname}/modules/{module-name}/{field-name}/`, with all items automatically converted to `kebab-casing`. For example, for the `value` field in our `ValueSetter` walkthrough, the SDK would generate an endpoint at the path `/modules/value-setter/value`.

Note that `ModuleRestApi` can't always generate endpoints for you. If it can't figure out how to generate an endpoint for a particular state value, it will simply skip it by default. If you want to override this behavior and throw a compiler error if endpoint generation fails, you can add the `#[rest_api(include)]` attribute.

### The `Spec` Generic

Modules are generic over a `Spec` type, which provides access to core rollup types. By being generic over the `Spec`, you ensure that you can easily change things like the `Address` format used by your module later on without rewriting your logic.

Key types provided by `Spec` include:
- `S::Address`: The address format used on the rollup.
- `S::Da::Address`: The address format of the underlying Data Availability layer.
- `S::Da::BlockHeader`: The block header type of the DA layer.

#### A Quick Tip on Schemas and Generics
If you parameterize your `CallMessage` or `Event` over `S` (for example, to include an address of type `S::Address`), you must add the `#[schemars(bound = "S: Spec")]` attribute on top your enum definition.
This is a necessary hint for `schemars`, a library that generates a JSON schema for your module's API. It ensures that your generic types can be correctly represented for external tools.

### `#[state]`, `#[module]`, and `#[id]` fields

-   **`#[id]`**: Every module must have exactly one `#[id]` field. The `ModuleInfo` macro uses this to store the module's unique, auto-generated identifier.
-   **`#[module]`**:  This attribute declares a dependency on another module. For example, if our `ValueSetter` needed to pay a fee, we could add `#[module] pub bank: sov_bank::Bank<S>`, allowing us to call `self.bank.transfer(...)` in our logic.
-   **`#[state]`**: This attribute marks a field as a state variable that will be stored in the database.

### State Types In-Depth

The SDK provides several state types, each for a different use case:
- **`StateValue<T>`**: Stores a single item of type T. We used this for `value` and `admin` variables. 
- **`StateMap<K, V>`**: Stores a key-value mapping.
- **`StateVec<T>`**: Stores an ordered list of items, accessible by index.

The generic types can be any deterministic Rust data structure, anything from a simple `u32` to a complex `BTreeMap`.

**Accessory State**: For each state type, there is a corresponding `AccessoryState*` variant (e.g., `AccessoryStateMap`). Accessory state is special: it can be read and written via the API, but it is **write-only** during a transaction. This makes it much cheaper to use for data that doesn't affect onchain logic, like indexing purchase histories for an off-chain frontend.

**Codecs**: By default, all state is serialized using Borsh. If you need to store a type from a third-party library that only supports serde, you can specify a different codec: `StateValue<ThirdPartyType, BcsCodec>`.

### The `Module` Trait and its Methods

The `Module` trait is the core of your application's onchain logic. The implementation you wrote in the walkthrough satisfies this trait's requirements.

Let's look at a simplified version of the trait definition to understand its components:

```rust
trait Module {
    /// The configuration needed to initialize the module, deserialized from `genesis.json`.
    type Config;

    /// A module-defined enum representing the actions a user can take.
    type CallMessage: Debug + BorshSerialize + BorshDeserialize + JsonSchema + UniversalWallet + Clone;

    /// A module-defined enum representing the events emitted by successful calls.
    type Event: Debug + BorshSerialize + BorshDeserialize + JsonSchema + 'static + core::marker::Send;

    /// `genesis` is called once when the rollup is deployed to initialize state.
    ///
    /// The logic here must be deterministic, but since it only runs once,
    /// efficiency is not a primary concern.
    fn genesis(
        &mut self,
        genesis_rollup_header: &<<Self::Spec as Spec>::Da as DaSpec>::BlockHeader,
        config: &Self::Config,
        state: &mut impl GenesisState<Self::Spec>,
    ) -> Result<(), Error>;


    /// `call` accepts a `CallMessage` and executes it, changing the module's state.
    fn call(
        &mut self,
        message: Self::CallMessage,
        context: &Context<Self::Spec>,
        state: &mut impl TxState<Self::Spec>,
    ) -> Result<CallResponse, Error>;
}
```

#### `genesis`
The `genesis` function is called once when the rollup is deployed. It uses the module's `Config` struct (defined as the associated `type Config`) to initialize the state. This `Config` is deserialized from the `genesis.json` file.

#### `call`
The `call` function provides the transaction processing logic. It accepts a structured `CallMessage` from a user and a `Context` containing metadata like the sender's address. If your `call` function returns an error, the SDK automatically reverts all state changes and discards any events, ensuring that transactions are atomic.

You can define the `CallMessage` to be any type you wish, but an enum is usually the best. Be sure to derive `borsh` and `serde` serialization (you can use our `#[serialize(Borsh, Serde])]` helper macro as in the example), as well as `schemars::JsonSchema` and `UniversalWallet`. This ensures your `CallMessage` is portable across different languages and frontends.

**A Note on Gas and Security**: Just like Ethereum smart contracts, modules accept inputs that are pre-validated by the chain. Your call method does not need to worry about authenticating the transaction sender. The SDK also automatically meters gas for state accesses. You only need to manually charge gas (using `self::charge_gas(...)` within your module logic) if your module performs heavy computation outside of state reads/writes.

### Events: Bridging On-Chain and Off-Chain Data

While call methods alter your module's state, **events** are how your module broadcasts those changes. They are the primary mechanism for streaming on-chain data to off-chain systems like databases and front-ends in real-time.

A key guarantee of the Sovereign SDK is that event emission is **atomic** with transaction execution—if a transaction reverts, so do its events. This ensures any off-chain system remains perfectly consistent with the on-chain state. The sequencer provides a robust websocket endpoint that streams sequentially numbered transactions along with their corresponding events. If a client disconnects, it can reliably resume the stream from the last transaction it processed, making it simple to build scalable and fault-tolerant off-chain data pipelines.

To implement events, you first define an `Event` enum, then emit it from your call method using `self.emit_event(state, <your_event_variant>)`.

**1. Define the Event**

```rust
// The event our module will emit after a successful action.
#[derive(Clone, Debug, PartialEq, Eq, JsonSchema)]
#[serialize(Borsh, Serde)]
#[serde(rename_all = "snake_case")]
pub enum Event {
    ValueUpdated(u32),
}

// Now, implement the `Module` trait with Event defined.
impl<S: Spec> Module for ValueSetter<S> {
    // ... existing code ...
    type CallMessage = CallMessage;

    // Plug in the Event type
    type Event = Event;
```

**2. Emit the Event**

Now, modify `set_value` in `module-system/module-implementations/examples/value-setter/src/call.rs` to emit `ValueUpdated`:

```rust
use sov_modules_api::EventEmitter;

impl<S: Spec> ValueSetter<S> {
    fn set_value(&mut self, new_value: u32, context: &Context<S>, state: &mut impl TxState<S>) -> Result<()> {

        // ... existing code ...
        self.value.set(&new_value, state)?;

        // Emit an event to record this change off-chain.
        self.emit_event(state, Event::ValueUpdated(new_value));

        Ok(())
    }
}
```

With these changes, every successful `set_value` transaction will emit an event that can be consumed by off-chain services.

### Error Handling

Modules use `anyhow::Result` for error handling, providing rich context that helps both developers and users understand what went wrong.

When your call method returns an Err, the SDK automatically reverts all state changes made during the transaction. This ensures that your module's logic is atomic.

```rust
impl<S: Spec> ValueSetter<S> {
    fn set_value(&mut self, new_value: u32, context: &Context<S>, state: &mut impl TxState<S>) -> Result<()> {
        // ... existing code ...

        // Check if the sender is admin
        if admin != *context.sender() {
            return Err(anyhow::anyhow!("Only the admin can set the value.").into());
        }

        // ... existing code ...
    }
```
For more details on error handling patterns, see the [Advanced Topics](3-5-advanced.md#error-handling) section.

### Next Step: Ensuring Correctness

You now have a deep understanding of how to define, implement, and structure a module. With this foundation, you're ready to test your module.

In the next section, **"Testing Your Module,"** we'll show you how to use the SDK's powerful testing framework to write comprehensive tests for your new module.