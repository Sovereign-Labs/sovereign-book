# Quickstart: Your First Module

This guide is about getting you to **write and deploy your own logic** on a rollup.

We'll start with a very basic [`ValueSetter`](https://github.com/Sovereign-Labs/sov-rollup-starter-wip/tree/main/examples/value-setter) module that's already included in the `sov-rollup-starter-wip`.

Our `ValueSetter` module currently stores a single number, and any user can update it. We want to ensure that only one user—the admin—has permission to update this number.

This requires four changes:
1.  **Add an** `admin` **field** to the module's state to store the admin address.
2.  **Create a configuration struct** so we can set the admin address when the rollup launches.
3.  **Initialize the** `admin`from the configuration struct in the `genesis` method, which sets up the module's initial state.
4.  **Add a check** in the `call` method to verify that the transaction sender is the admin.

Let's get started.

## Step 1: Understand the Starting Point

First, navigate to the `value-setter` module in the starter repository and open the `src/lib.rs` file.

```bash
# From the sov-rollup-starter root
cd examples/value-setter/
```

The initial code is minimal. It defines the module's structure and a `call` method that lets anyone set the value.

Here’s the simplified `lib.rs` that we'll start with:

```rust
// In examples/value-setter/src/lib.rs

#[derive(Clone, ModuleInfo, ModuleRestApi)]
pub struct ValueSetter<S: Spec> {
    #[id]
    pub id: ModuleId,

    /// Holds the value
    #[state]
    pub value: StateValue<u32>,
}

#[derive(Clone, Debug, PartialEq, Eq, JsonSchema, UniversalWallet)]
#[serialize(Borsh, Serde)]
#[serde(rename_all = "snake_case")]
pub enum CallMessage {
    SetValue(u32),
}

impl<S: Spec> Module for ValueSetter<S> {
    type Spec = S;
    type Config = (); // No configuration yet!
    type CallMessage = CallMessage;
    type Event = ();

    // The `call` method handles incoming transactions.
    // Notice it doesn't check *who* is calling.
    fn call(&mut self, msg: Self::CallMessage, _context: &Context<S>, state: &mut impl TxState<S>) -> Result<()> {
        match msg {
            CallMessage::SetValue(new_value) => {

                self.value.set(&new_value, state)?;

                Ok(())
            }
        }
    }
}
```

## Step 2: Implement the Admin Logic

Now, let's secure our module. We'll perform the four edits we outlined earlier.

### a) Add the `admin` State Variable

First, we need a place to store the admin's address. We'll add a new `admin` field to the `ValueSetter` struct and mark it with the `#[state]` attribute.

```rust
// In examples/value-setter/src/lib.rs

#[derive(Clone, ModuleInfo, ModuleRestApi)]
pub struct ValueSetter<S: Spec> {
    // ... existing code ...

    /// The new state value to hold the address of the admin.
    #[state]
    pub admin: StateValue<S::Address>,
}
```

### b) Define a Configuration Struct

Next, we need a way to tell the module who the admin is when the rollup first starts. We do this by defining a `Config` struct. The SDK will automatically load data from a `genesis.json` file into this struct.

```rust
// In examples/value-setter/src/lib.rs

// Add the module's configuration, read from genesis.json
#[derive(Clone, Debug, PartialEq, Eq)]
#[serialize(Serde)]
#[serde(rename_all = "snake_case")]
pub struct ValueSetterConfig<S: Spec> {
    pub admin: S::Address,
}
```

### c) Initialize the Admin at Genesis

With our `Config` struct defined, we can now implement the `genesis` method. This function is called once when the rollup is launched. It takes the `config` as an argument and uses it to set the initial state.

We also need to tell the `Module` implementation to use our new `ValueSetterConfig`.

```rust
// In examples/value-setter/src/lib.rs

// ... existing code ...

impl<S: Spec> Module for ValueSetter<S> {
    type Spec = S;
    type Config = ValueSetterConfig<S>; // Use the new config struct
    type CallMessage = CallMessage;
    type Event = ();

    // `genesis` initializes the module's state. Here, we set the admin address.
    fn genesis(&mut self, _header: &<S::Da as sov_modules_api::DaSpec>::BlockHeader, config: &Self::Config, state: &mut impl GenesisState<S>) -> Result<()> {
        self.admin.set(&config.admin, state)?;
        Ok(())
    }

    fn call(&mut self, msg: Self::CallMessage, context: &Context<S>, state: &mut impl TxState<S>) -> Result<()> {
// ... existing code ...
```

### d) Add the Admin Check in `call`

The final piece. We'll modify the `call` method to read the `admin` address from state and compare it to the transaction sender. If they don't match, the transaction fails.

```rust
// In examples/value-setter/src/lib.rs
// ... existing code ...

    fn call(&mut self, msg: Self::CallMessage, context: &Context<S>, state: &mut impl TxState<S>) -> Result<()> {
        match msg {
            CallMessage::SetValue(new_value) => {
                // Read the admin's address from state.
                let admin = self.admin.get_or_err(state)??;

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

## Step 3: Configure the Genesis State

Our `genesis` method reads the admin's address from a configuration file. We need to provide that value in `configs/mock_da/genesis.json`.

The SDK automatically deserializes this JSON into our `ValueSetterConfig` struct (since we plugged in said struct as the `Config` associated type of our module) when the rollup starts.

```json
// In sov-rollup-starter/configs/mock_da/genesis.json
{
  // ... other module configs
  "value_setter": {
    "admin": "0x9b08ce57a93751aE790698A2C9ebc76A78F23E25"
  }
}
```

Previously, the `value_setter` field was `null`. Now, we've given it the data our module needs to initialize the admin address.

## How is the Module Integrated?

You might be wondering how the rollup knows about the `value-setter` module in the first place. In the `sov-rollup-starter`, we've already "wired it up" for you to keep this quickstart focused on module logic.

For your own future modules, the process involves:
1.  Adding the module crate to the workspace in the root `Cargo.toml`.
2.  Adding it as a dependency to the core logic in `crates/stf/Cargo.toml`.
3.  Adding the module as a field on the `Runtime` struct in `crates/stf/src/runtime.rs`.

You can remove `value-setter` from these files to see what it's like to build and integrate a module from scratch.

## Step 4: Build, Run, and Interact!

Now let's see your logic in action.

1.  **Build and Run the Rollup:** From the root directory, start the rollup.

    ```bash
    cargo run
    ```

2.  **Query the Initial State:** In another terminal, use `curl` to check the initial value. It should be `null` because our `genesis` method only sets the `admin`, not the `value`.

    ```bash
    curl http://127.0.0.1:12346/modules/value-setter/state/value
    # Expected output: {"value":null}
    ```

3.  **Submit a Transaction:** Now, let's change the value. We'll edit the example js script in starter to call our module.
    *   Open the `examples/starter-js/src/index.ts` file.
    *   The `signer` in this script corresponds to the `admin` address we set in `genesis.json`.
    *   Find the `callMessage` variable and replace it with a call to your `value_setter` module.

    ```ts
    // In sov-rollup-starter/examples/starter-js/src/index.ts

    // Replace the existing call message with this one:
    const callMessage: RuntimeCall = {
        value_setter: {   // The module's name in the Runtime struct
            set_value: 99,  // The CallMessage variant (in snake_case) and its new value
        },
    };
    ```

    *   Install js dependencies, and run the script to send the transaction:

    ```bash
    # From the sov-rollup-starter/examples/starter-js directory
    npm install
    npm run start
    ```

4.  **Verify the Change:** Now for the "Aha!" moment. Query the state again:

    ```bash
    curl http://127.0.0.1:12346/modules/value-setter/state/value
    # Expected output: {"value":99}
    ```

**Congratulations!** You have successfully written and interacted with your own custom logic on a Sovereign SDK rollup! 