# Getting Started

## Overview

This guide provides a starting point for building rollups with the Sovereign SDK. 

It includes everything you need to create a rollup with customizable modules, REST API for state queries, TypeScript SDK for submitting transactions, WebSocket endpoints to subscribe to transactions and events, built-in token management, and much more.

## Prerequisites

Before you begin, ensure you have the following installed:

- **Rust**: 1.88.0 or later
  - Install via [rustup](https://rustup.rs/): `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`
  - The project will automatically install the correct version via `rust-toolchain.toml`
- **Node.js**: 18.0 or later (for Typescript client)
  - Install via [official website](https://nodejs.org/en/download)
- **Git**: For cloning the repository

## Running with Mock DA

### 1. Clone the starter repository and navigate to the rollup directory:

```bash
git clone https://github.com/Sovereign-Labs/sov-rollup-starter.git
cd sov-rollup-starter/crates/rollup/
```

### 2. (Optional) Clean the database for a fresh start:

```bash
make clean-db
```

### 3. Start the rollup node:

```bash
cargo run --bin node
```

### Explore the REST API endpoints via Swagger UI

The rollup starter includes several built-in modules: Bank (for token management), Paymaster, Hyperlane, and more. You can query any state item in these modules:

```bash
open http://localhost:12346/swagger-ui/#/ 
```

### Example: Query the Example Module's state value:

```bash
curl -X 'GET' \
  'http://0.0.0.0:12346/modules/example-module/state/value' \
  -H 'accept: application/json'
```

For now, you should just see null returned for the value state item, as the item hasn't been initialized:

```bash
{"data":{"value":null},"meta":{}}
```

## Programmatic Interaction with Typescript

### Set up the Typescript client:

```bash
cd ../../js # Navigate back up to the right directory
npm install 
```

### The Typescript script demonstrates the complete transaction flow:

```js
// 1. Initialize rollup client
const rollup = await createStandardRollup({ // defaults to http://localhost:12346, or pass url: "<custom-endpoint>"
  context: {
    defaultTxDetails: {
      max_priority_fee_bips: 0,
      max_fee: "100000000",
      gas_limit: null,
      chain_id: 4321, // Must match chain_id in constants.toml
    },
  },
});

// 2. Initialize signer
const privKey = "0d87c12ea7c12024b3f70a26d735874608f17c8bce2b48e6fe87389310191264";
let signer = new Secp256k1Signer(privKey, chainHash);

// 3. Create a transaction (call message)
let createTokenCall: RuntimeCall = {
  bank: {
    create_token: {
      admins: [],
      token_decimals: 8,
      supply_cap: 100000000000,
      token_name: "Example Token",
      initial_balance: 1000000000,
      mint_to_address: signerAddress, // derived from privKey above (can be any valid address)
    },
  },
};

// 4. Send transaction
let tx_response = await rollup.call(createTokenCall, { signer });
```

### Run the script:
```bash
npm run start 
```

You should see a transaction soft-confirmation with events:
```bash
Tx sent successfully. Response:
{
  data: {
    id: '0xbfe14371219807b236c5c719ea85be63174fe0c673e8b229e4913e6f6273a5a0',
    events: [
      {
        type: 'event',
        number: 0,
        key: 'Bank/TokenCreated',
        value: {
          token_created: {
            token_name: 'Example Token',
            coins: {
              amount: '1000000000',
              token_id: 'token_10jrdwqkd0d4zf775np8x3tx29rk7j5m0nz9wj8t7czshylwhnsyqpgqtr9'
            },
            mint_to_address: { user: '0x9b08ce57a93751ae790698a2c9ebc76a78f23e25' },
            minter: { user: '0x9b08ce57a93751ae790698a2c9ebc76a78f23e25' },
            supply_cap: '100000000000',
            admins: []
          }
        },
        module: { type: 'moduleRef', name: 'Bank' },
        tx_hash: '0xbfe14371219807b236c5c719ea85be63174fe0c673e8b229e4913e6f6273a5a0'
      }
    ],
    receipt: { result: 'successful', data: { gas_used: [ 21119, 21119 ] } },
    status: 'submitted'
  },
  meta: {}
}
```

### Subscribe to events from the sequencer:

You can also subscribe to events from the sequencer (you need to uncomment the subscription code blocks [in the script](js/src/index.ts#L53)):

```js
// Subscribe to events
async function handleNewEvent(event: any): Promise<void> {
  console.log(event);
}
const subscription = rollup.subscribe("events", handleNewEvent);

// Unsubscribe
subscription.unsubscribe();
```

### Interacting with different modules

To interact with different modules, simply change the call message. The top-level key corresponds to the [module's variable name in the runtime](/crates/stf/src/runtime.rs#L85), and the nested key is the [CallMessage](fix-link) enum variant in snake_case:

```js
// Example: Call the ExampleModule's SetValue method
let setValueCall: RuntimeCall = {
  example_module: {  // Must match Runtime field name of the module
    set_value: 10  
  },
};
```

This transaction would set the ExampleModule's state value to 10. Try setting the [example file's call message](js/src/index.ts#L39) to the expression above and re-running the script. Then verify that the ExampleModule's value changed using [the curl command](#example-query-the-example-modules-state-value) we showed earlier. 

This time, the curl command should return:
```bash
{"data":{"value":10},"meta":{}}
```

### Learn more

To learn more about building with Sovereign SDK, experiment with the [ExampleModule](/crates/example-module/src/lib.rs). For a deeper understanding of the abstractions, continue to the Writing Your Application chapter.