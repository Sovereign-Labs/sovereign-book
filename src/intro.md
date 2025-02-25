# The Sovereign SDK Book

The Sovereign SDK is a batteries-included framework for building onchain
applications.

<a href="https://discord.gg/kbykCcPrcA" ><img alt="Discord" src="https://img.shields.io/discord/1050059327626555462?label=discord"/></a>

<img src="https://github.com/Sovereign-Labs/sovereign-sdk/blob/nightly/assets/banner.jpg?raw=true" style="border-radius: 10px">

## Why Rollups?

As a developer, building your application as a rollup has several advantages:

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

## Why Sovereign?

The Sovereign SDK is the most flexible framework for building rollups. Unlike
other rollup frameworks, the Sovereign SDK supports rollups without a settlement
layer. That means that you can deploy your rollup anywhere - including on
Bitcoin and Celestia. The SDK also provides top-tier scalability and a seamless
user experience, all without sacrificing flexibility. Teams are already using
the Sovereign SDK to build...

- An EVM chain on Bitcoin
- A MoveVM chain on Celestia
- Appchains on Solana

... and much more.

## How it Works

As a developer, you write the business logic of your rollup in Rust and the SDK
handles all of the complexity of creating a rollup on their behalf. Under the
hood, the SDK compiles the chain's business logic to a zero-knowledge circuit,
which it uses to prove correct execution (if the rollup is running in "zk mode")
or to resolve disputes about execution (if the rollup is running in "optimistic
mode"). It also generates a complete _full node_ implementation which can
reproduce the state of the blockchain and serve data to users.

Once the rollup is deployed, users post their _transactions_ onto an underlying
blockchain called a _Data Availability Layer_ ("DA Layer") for ordering. After
transactions are ordered, the _full nodes_ of the rollup execute them to compute
the new rollup _state_.

Finally, specialized actors called "provers" or "attesters" generate a proof
that the new rollup state was computed correctly and post the proof back onto
the DA layer. This enables clients of the rollup to verify claims about the
rollup state without running a full node for themselves.
