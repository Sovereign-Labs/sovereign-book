# Advanced Topics

This section covers advanced module development features that go beyond basic functionality. While the core module implementation handles state management and transaction processing, you may need these additional capabilities for production use cases.

All features in this section are optional. Start with the basic module implementation and add these capabilities as your requirements grow.

## Hooks

In addition to `call`, modules may _optionally_ implement `Hooks`. Hooks can run at
the begining and end of every rollup block and every transaction. `BlockHooks`
are great for taking actions that need to happen before or after any
transaction executes in a block - but be careful, no one pays for the
computation done by `BlockHooks`, so doing any heavy computation can make your
rollup vulnerable to DOS attacks.

`TxHooks` are useful for checking invariants, or to allow your module to monitor actions
being taken by other modules. Unlike `BlockHooks`, `TxHooks` are paid for by the
user who sent each transaction.

The `FinalizeHook` is great for doing indexing. It can only modify
`AccessoryState`, which makes it cheap to run but means that the results will
only be visible via the API.

Using the hooks is somewhat unusual - most applications only need to modify
their state in response to user actions - but it's a powerful tool in some
cases. See the documentation on
[`BlockHooks`](fix-link/crates/module-system/sov-modules-api/src/hooks.rs#L76)
and
[`TxHooks`](fix-link/crates/module-system/sov-modules-api/src/hooks.rs#L12)
and
[`FinalizeHook`](fix-link/crates/module-system/sov-modules-api/src/hooks.rs#L120)
more details.

## Error Handling

### When to Panic vs Return Errors

**Panic when:**
- You encounter a bug that indicates broken invariants
- The error is unrecoverable and continuing would compromise state integrity

When you panic, the rollup will shut down. This is correct for bugs that could corrupt your state.

**Return errors when:**
- User input is invalid
- Business logic conditions aren't met (insufficient balance, unauthorized access, etc.)
- Any expected failure condition

Transaction errors automatically revert all state changes.

### Writing Error Messages

Your error messages serve both end users and developers. Use `anyhow` with context to provide meaningful errors:

```rust
use anyhow::{Context, Result};

fn transfer(&self, from: &S::Address, to: &S::Address, token_id: &TokenId, amount: u64, state: &mut impl TxState<S>) -> Result<()> {
    let balance = self.balances
        .get(&(from, token_id), state)
        .context("Failed to read sender balance")?
        .unwrap_or(0);
    
    if balance < amount {
        // User-facing error message
        return Err(anyhow::anyhow!("Insufficient balance: {} < {}", balance, amount));
    }
    
    let new_balance = balance - amount;
    
    // Add context for debugging when operations fail
    self.balances
        .set(&(from, token_id), &new_balance, state)
        .with_context(|| format!("Failed to update balance for {} token {}", from, token_id))?;
    
    // ... rest of transfer logic
    Ok(())
}
```

Transaction reverts are normal and expected - log them at `debug!` level if needed for debugging, not as warnings or errors.

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

By leveraging Hooks, robust error handling, and custom APIs, you can build sophisticated, production-grade modules that are both powerful and easy to operate.

With a deep understanding of module implementation, you may next want to optimize your rollup's performance. The next section on **"Understanding Performance"** will dive into state access patterns and cryptographic considerations that can significantly impact your application's throughput.