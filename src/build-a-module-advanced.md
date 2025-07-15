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

## Native-Only Code

Some functionality should only run natively, not in the zkVM during proof generation. This includes:
- Custom REST APIs and RPC methods
- Metrics and logging
- Integration with external services
- Debugging and development tools

Any code that shouldn't be part of state transition verification must be gated with `#[cfg(feature = "native")]`:

```rust
#[cfg(feature = "native")]
impl<S: Spec> MyModule<S> {
    // This code only compiles natively, not in zkVM
    pub fn debug_state(&self, state: &impl StateAccessor<S>) {
        let total_items = self.items.len(state);
        println!("Total items: {}", total_items);
    }
}
```

This ensures that:
- zkVM execution remains deterministic and efficient
- Proof generation doesn't include unnecessary code

## Custom APIs 

Using the native-only pattern, you can add custom REST endpoints and RPC methods to your module.

### Adding Custom REST APIs

You can easily add custom APIs to your module by implementing the
`HasCustomRestApi` trait. This trait has two methods - one which actually
implements the routes, and an optional one which provides an `OpenApi` spec. You
can see a good example in the `Bank` module:

```rust
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
some examples in the [`Evm` module](fix-link).

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