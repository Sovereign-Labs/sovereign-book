# Advanced Topics

This section covers advanced module development features that go beyond basic functionality. 

Need to run logic on every block? Want to build custom APIs or integrate with off-chain services? Need configurable delays to reduce MEV for your application? You'll find the answers here.

Each of these features is optional, designed to be adopted as your application's needs evolve.

## Hooks: Responding to On-Chain Events

While the `call` method allows your module to react to direct user transactions, sometimes 
you need your module to execute logic in response to broader onchain events. This is where `Hooks` come in. 
They allow your module to "hook into" the lifecycle of a block or transaction, enabling 
powerful automation and cross-module interactions.

### `BlockHooks`: Running Logic at Block Boundaries

`BlockHooks` are triggered at the beginning and end of every block. They are ideal for logic that 
needs to run periodically, independent of any specific transaction. For example, you could use 
a `BlockHook` to:

- Distribute rewards once per block.
- Update funding rate looking at the number of open positions at the end of every N blocks.

**A word of caution**: BlockHook computation is not paid for by any single user, so it's a "public good" of your rollup. 
Be mindful of performance here; heavy computation in a BlockHook can make your rollup vulnerable to Denial-of-Service (DoS) attacks.

### TxHooks: Monitoring All Transactions

TxHooks run before and after every single transaction processed by the rollup. This makes them perfect for:

- **Global Invariant Checks**: Ensuring a global property (like total supply of a token) is never violated by any module.
- **Monitoring and Reactions**: Allowing a compliance module to monitor all transfers and flag suspicious activity.

Unlike `BlockHooks`, the gas for `TxHooks` is paid by the user who submitted the transaction.

### `FinalizeHook`: Cheap Off-Chain Indexing

The `FinalizeHook` runs at the very end of a block's execution and can only write to `AccessoryState`. This makes it 
cheap to run and perfect for storing data that are only meant to be read by off-chain APIs, not used by on-chain logic.

See more documentation on hooks here:
- [`BlockHooks`](fix-link/crates/module-system/sov-modules-api/src/hooks.rs#L76)
- [`TxHooks`](fix-link/crates/module-system/sov-modules-api/src/hooks.rs#L12)
- [`FinalizeHook`](fix-link/crates/module-system/sov-modules-api/src/hooks.rs#L120)

## Error Handling: User Errors vs. System Bugs

In a blockchain context, handling failure correctly is critical. Your module must clearly distinguish between two types of failures: expected **user errors** (which should gracefully revert a transaction) and unexpected **system bugs** (which may require halting the chain to prevent state corruption). The Sovereign SDK provides a clear pattern for this distinction.

### 1. User Errors: Returning `anyhow::Result`

For all expected, business-logic-level failures, your `call` method should return an `Err`. These are the errors you anticipate, such as a user attempting to transfer more tokens than they own, calling a method without the proper permissions, or providing invalid parameters.

When you return an `Err`, the SDK automatically reverts all state changes from the transaction. The goal is to safely reject the invalid transaction while providing a clear error message to the user and/or developer.

The `anyhow` crate provides several convenient macros for this. While you can always construct an error with `anyhow::anyhow!()`, the `bail!` and `ensure!` macros are generally preferred for their conciseness.

*   **`bail!(message)`**: Immediately returns an `Err`. It's a direct shortcut for `return Err(anyhow::anyhow!(message))`.
*   **`ensure!(condition, message)`**: Checks a condition. If it's false, it returns an `Err` with the given message. This is perfect for validating inputs and permissions at the start of a function.

Here’s how they look in practice, using the `Bank` module as an example:

```rust
// From the Bank module's `create_token` method
fn create_token(
    // ...
) -> Result<TokenId> {
    // Using `ensure!` to validate an input parameter.
    anyhow::ensure!(
        token_decimals <= MAX_DECIMALS,
        "Too many decimal places."
    );

    // Using `bail!` to return an error after a more complex check.
    if initial_balance > supply_cap {
        bail!(
            "Initial balance {} is greater than the supply cap {}",
            initial_balance,
            supply_cap
        );
    }
    // ...
    Ok(token_id)
}
```

Because transaction reverts are a normal part of operation, they should be logged at a `debug` level if necessary, not as warnings or errors.

### 2. System Bugs: `panic!`

A `panic!` is an emergency stop. It should only be used for critical, unrecoverable bugs where a core assumption or invariant of your system has been violated.

*   **When:** An impossible state is reached (e.g., total supply becomes negative).
*   **What it does:** Shuts down the rollup node to prevent state corruption.
*   **Goal:** Alert the node operator to a critical software bug that needs immediate attention.

Use `panic!` as your last line of defense. It signals that your module's integrity is compromised and continuing execution would be dangerous.

## Native-Only Code

Some functionality should only run natively on the full nodes (and sequencer), not in the zkVM during proof generation. This is a critical concept for separating verifiable on-chain logic from off-chain operational tooling.

Any code that is not part of the core state transition must be gated with `#[cfg(feature = "native")]`:

```rust
#[cfg(feature = "native")]
impl<S: Spec> MyModule<S> {
    // This code only compiles natively, not in zkVM
    pub fn debug_state(&self, state: &impl StateAccessor<S>) {
        // ...
    }
}
```
This ensures that your zk-proofs remain small and your onchain logic remains deterministic. Common use cases for native-only code include:

- Custom REST APIs and RPC methods
- Metrics and logging integration
- Debugging tools
- Integrations with external services

## Transaction Prioritization and MEV Mitigation

For latency-sensitive financial applications, managing transaction order and mitigating Maximum Extractable Value (MEV) is critical. The Sovereign SDK provides a powerful, sequencer-level tool to combat toxic orderflow by allowing developers to introduce fine-grained processing delays for specific transaction types.

This is a powerful technique for applications like on-chain Central Limit Orderbooks (CLOBs). By introducing a small, artificial delay on aggressive "take" orders, a rollup can implicitly prioritize "cancel" orders. This gives market makers a crucial window to pull stale quotes before they can be exploited by low-latency arbitrageurs, leading to fairer and more liquid markets.

This functionality is implemented via the `get_transaction_delay_ms` method on your `Runtime` struct. Because this is a sequencer-level scheduling feature and not part of the core state transition logic, it must be gated behind the `native` feature flag.

The method receives a decoded `CallMessage` and returns the number of milliseconds the sequencer should wait before processing it. A return value of `0` means the transaction should be processed immediately.

### Example: Prioritizing Cancels in a CLOB

```rust
// In your-rollup/stf/src/runtime.rs

// In the `impl<S> sov_modules_stf_blueprint::Runtime<S> for Runtime<S>` block:

#[cfg(feature = "native")]
fn get_transaction_delay_ms(&self, call: &Self::Decodable) -> u64 {
    // `Self::Decodable` is the auto-generated `RuntimeCall` enum for your runtime.
    // It has one variant for each module in your `Runtime` struct.
    match call {
        // Introduce a small 10ms delay on all "take" orders to give
        // market makers time to cancel stale orders.
        // (Here, `Clob` is the variant corresponding to the `clob` field in your `Runtime` struct,
        // and `PlaceTakeOrder` is the variant of the `clob` module's `CallMessage` enum.)
        Self::Decodable::Clob(clob::CallMessage::PlaceTakeOrder { .. }) => 50,

        // All other CLOB operations, like placing or cancelling "make" orders,
        // are processed immediately with zero delay.
        Self::Decodable::Clob(..) => 0,
        
        // All other transactions in other modules are also processed immediately.
        _ => 0,
    }
}
```

This feature gives you precise control over your sequencer's processing queue, enabling sophisticated MEV mitigation strategies without altering your core onchain business logic.

## Adding Custom REST APIs

You can easily add custom APIs to your module by implementing the
`HasCustomRestApi` trait. This trait has two methods - one which actually
implements the routes, and an optional one which provides an `OpenApi` spec. You
can see a good example in the `Bank` module:

```rust
#![cfg(feature = "native")]
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

## Legacy RPC Support

In addition to custom RESTful APIs, the Sovereign SDK lets you create JSON-RPC
methods. This is useful to provide API compatibility with existing chains like
Ethereum and Solana, but we recommend using REST APIs whenever compatibility
isn't a concern.

To implement RPC methods, simply annotate an `impl` block on your module with
the `#[rpc_gen(client, server)]` macro, and then write methods which accept an
`ApiStateAcessor` as their final argument and return an `RpcResult`. You can see
some examples in the [`Evm` module](fix-link).

```rust
#![cfg(feature = "native")]
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

### Mastering Your Module

You are now equipped with the full suite of tools for building sophisticated, production-grade modules. 
By leveraging Hooks for automation, robust error handling for reliability, and custom APIs for rich off-chain 
experiences, you can create applications that are both powerful and easy to operate.

With a deep understanding of module implementation, you're ready to shift your focus to a critical aspect of any rollup: performance. . The next section, **"Understanding Performance,"** will dive into  patterns and considerations that can significantly impact your application's latency and throughput.