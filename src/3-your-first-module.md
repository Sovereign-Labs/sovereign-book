# Quickstart: Your First Module

In the last chapter, you successfully ran the starter rollup and verified your environment. Now, it's time for the most exciting part: writing and deploying your *own* logic.

This chapter is designed for speed. We'll guide you through the shortest possible path to creating, integrating, and interacting with a custom module. We'll skip the deep dives and focus on getting you a "win" as quickly as possible.

We'll be working with the `ValueSetter` module that already exists in the `sov-rollup-starter`, but we'll explain every step as if you were building it from scratch.

## Step 1: The Module's Logic

First, navigate to the `value-setter` module in the starter repository and open the `src/lib.rs` file.

```bash
# From the sov-rollup-starter root
cd examples/value-setter/
```

The file contains a complete module implementation. Let's look at the essential parts.

### a) Define the Module Struct

First, we define the module's structure and the state it will manage. This struct tells the SDK what data to store onchain.

```rust
// In examples/value-setter/src/lib.rs
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
    #[state]
    pub admin: StateValue<S::Address>,
}
```

### b) Define Types and Logic

With the struct in place, we define its `Config`, `CallMessage`, and the main `Module` logic.

```rust
// In examples/value-setter/src/lib.rs

// 1. The module's configuration, read from genesis.json
#[derive(Clone, Debug, PartialEq, Eq)]
#[serialize(Borsh, Serde)]
pub struct ValueSetterConfig<S: Spec> {
    pub admin: S::Address,
}

// 2. The single action a user can perform: setting the value.
#[derive(Clone, Debug, PartialEq, Eq, JsonSchema, UniversalWallet)]
#[serialize(Borsh, Serde)]
pub enum CallMessage {
    SetValue(u32),
}

// 3. The module's core logic
impl<S: Spec> Module for ValueSetter<S> {
    type Spec = S;
    type Config = ValueSetterConfig<S>;
    type CallMessage = CallMessage;
    type Event = (); // We'll skip events for now

    // `genesis` initializes the module's state. Here, we set the admin address.
    fn genesis(&mut self, _header: &<S::Da as sov_modules_api::DaSpec>::BlockHeader, config: &Self::Config, state: &mut impl GenesisState<S>) -> Result<()> {
        self.admin.set(&config.admin, state)?;
        Ok(())
    }

    // `call` handles incoming user transactions.
    fn call(&mut self, msg: Self::CallMessage, context: &Context<S>, state: &mut impl TxState<S>) -> Result<()> {
        match msg {
            CallMessage::SetValue(new_value) => {
                let admin = self.admin.get(state)?.unwrap();

                // Ensure the sender is the admin.
                anyhow::ensure!(admin == *context.sender(), "Only the admin can set the value.");

                // If the check passes, update the state.
                self.value.set(&new_value, state)?;
                Ok(())
            }
        }
    }
}
```

This is the core of our module. It defines a single state variable `value`, an `admin` that can change it, and a `SetValue` method.

## Step 2: Add the Module to the Runtime

Now that we have our module's logic, we need to tell the rollup to include it. This involves editing three files: the root `Cargo.toml`, the state transition function's `Cargo.toml`, and the `runtime.rs` file itself.

### a) Add the Module to the Workspace

First, make your module crate visible to the rest of the rollup. In a standard Rust workspace, this is done in the root `Cargo.toml` file by adding your module's path to the `[workspace.dependencies]` section.

```toml
# In sov-rollup-starter/Cargo.toml
[workspace.dependencies]
# ...
value-setter = { path = "examples/value-setter", version = "0.1.0" }
```

This tells Cargo where to find your module's code and allows other crates in the workspace to refer to it.

### b) Add the Dependency to the Rollup's Core Logic

Next, add the `value-setter` crate as a dependency for the rollup's state transition function, which lives in `crates/stf/`.

```toml
# In sov-rollup-starter/crates/stf/Cargo.toml
[dependencies]
# ... existing dependencies
value-setter = { workspace = true }
```
The `workspace = true` syntax tells Cargo to inherit the path and version from the root `Cargo.toml` we just edited. This practice keeps all dependency versions consistent across the project.

### c) Add the Module to the `Runtime` Struct

Finally, add `value_setter` as a field to the `Runtime` struct in `crates/stf/src/runtime.rs`. This is the final step that actually incorporates the module into the rollup.

```rust
// In sov-rollup-starter/crates/stf/src/runtime.rs
#[derive(Genesis, DispatchCall, MessageCodec)]
pub struct Runtime<S: Spec> {
    // ... other modules
    pub value_setter: value_setter::ValueSetter<S>,
}
```

## Step 3: Configure Genesis State

The final integration step is to tell the rollup how to initialize our module at launch.

Back in `Step 1`, we defined a `Config` struct for our module:

```rust
// From examples/value-setter/src/lib.rs
pub struct ValueSetterConfig<S: Spec> {
    pub admin: S::Address,
}
```

The `genesis` method of our module uses this struct to set its initial state. Now, we provide the actual values for this configuration in the `genesis.json` file. The SDK automatically deserializes this JSON into our `Config` struct when the rollup starts.

```json
// In sov-rollup-starter/configs/mock_da/genesis.json
{
  // ... other module configs
  "value_setter": {
    "admin": "sov1h6t82p7my6k02tcz9a27nnsyd9chg490rwn3r09e8k02y5qew2qsl2a4vg"
  }
}
```

*(Note: In the starter repository, all these integration steps are already done for you, but it's important to understand the process for when you build your own modules.)*

## Step 4: Build, Run, and Interact!

This is the payoff. Let's see your module in action.

1.  **Build and Run the Rollup:** From the root directory, start the node.

    ```bash
    cd ../..  # Navigate back from the value-setter example
    cargo run --bin node
    ```

2.  **Query the Initial State:** In another terminal, use `curl` to check the initial value. It should be `null` because our `genesis` method only sets the `admin`, not the `value`.

    ```bash
    curl -X GET 'http://127.0.0.1:12346/modules/value-setter/value'
    # Expected output: {"data":{"value":null},"meta":{}}
    ```

3.  **Submit a Transaction:** Now, let's change the value. We'll edit the Typescript client to call our new module.
    *   Open the `js/src/index.ts` file.
    *   Find the `createTokenCall` variable and replace it with a call to your `value_setter` module. The address used by the `signer` in the script is the same one we configured as the `admin` in `genesis.json`.

    ```ts
    // In sov-rollup-starter/js/src/index.ts

    // Replace the existing call message with this one:
    let call: RuntimeCall = {
      value_setter: {   // The module's name in the Runtime struct
        set_value: 99,  // The CallMessage variant and its new value
      },
    };
    ```

    *   Run the script to send the transaction:

    ```bash
    # From the sov-rollup-starter/js directory
    npm run start
    ```

4.  **Verify the Change:** Now for the "Aha!" moment. Query the state again:

    ```bash
    curl -X GET 'http://127.0.0.1:12346/modules/value-setter/value'
    # Expected output: {"data":{"value":99},"meta":{}}
    ```

**Congratulations!** You have successfully built, integrated, and interacted with your own custom logic on a Sovereign SDK rollup. You've completed the core developer loop.

### What's Next?

This quickstart intentionally skipped over some important concepts like events, comprehensive error handling, and testing. To build a truly robust and feature-rich application, you'll need to understand these in more detail.

The next chapter, **"Building for Production,"** will revisit the `ValueSetter` module and explore the deeper concepts and best practices required to take your module from a simple example to a production-ready component.