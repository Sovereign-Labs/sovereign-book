# The Sovereign SDK Book

<a href="https://discord.gg/kbykCcPrcA" ><img alt="Discord" src="https://img.shields.io/discord/1050059327626555462?label=discord"/></a>

<img src="https://github.com/Sovereign-Labs/sovereign-sdk/blob/nightly/assets/banner.jpg?raw=true" style="border-radius: 10px">

The Sovereign SDK is a batteries-included framework for building blockchain
applications. By building with the SDK, you get...

- A high performance blockchain "backend" for your app
- Best-in-class user experience including gasless transactions, embedded wallet
  support, and human-readable transactions
- `REST`ful APIs for querying and indexing your application state and events
- JavaScript and TypeScript SDKs for interacting with your app's backend
- Support for web, and (coming soon) hardware wallets

Using the Sovereign SDK, you get complete control over your application. There
are no hard limits on gas usage or transaction size, and _everything_ about your
app is fully customizable.

At the same time, developing on the SDK is easy. We provide powerful default
configurations so that everything "just works" out of the box with minimal
configuration.

## How it Works

Under the hood, the Sovereign SDK generates a dedicated rollup for your
application. Compared to deploying a smart contract, running your app as a
custom rollup gives a bunch of advantages:

1. **Dedicated throughput:** your users won't pay more just because another app
   is generating a lot of transactions.
1. **Scalability:** Sovereign SDK nodes scale seamlessly to thousands of
   transactions per second on commodity hardware, and can achieve substantially
   higher throughput on optimized hardware.
1. **Incentive alignment:** the vast majority of the rollup fees can be
   distributed to users and developers of the rollup, rather than subsidizing
   token holders on L1.
1. **MEV mitigation:** since you have full control over your rollup logic, you
   can design your protocol to minimize MEV and capture the portions that can't
   be eliminated.
1. **Flexibility:** rollups enable you to express whatever logic you want,
   without worrying about the needs of other applications. Enable cutting edge
   EIPs and account abstraction, or ditch the EVM entirely and build an
   app-specific chain. **With a rollup, you're in the driver's seat.**
