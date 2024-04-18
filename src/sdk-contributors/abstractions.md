# Main Abstractions

One of the most important principles in the Sovereign SDK is modularity. We believe strongly in separating
rollups into their component parts and communicating through abstract interfaces. This allows us to iterate
more quickly (since components are unaware of the implementation details of other components), and it also
allows us to reuse components in contexts which are often quite different from the ones in which they were
orginally designed. 

In this chapter, we'll give a brief overview of the core abstractions of the Sovereign SDK


## Native vs. ZK Execution

Perhaps the most fundamental abstraction in Sovereign is the separation between `"native"` code execution
(which computes a new rollup state) and zero-knowledge _verification_ of that state. Native execution is
the experience you're used to. In native execution, you have full access to networking, disk, etc. In 
native mode, you typically trust data that you read from your own database, but not data that comes over
the network. 

Zero-knowledge execution looks similar. You write normal-looking Rust code to do CPU and memory
operations - but under the hood, the environment is alien. In zero-knowledge execution, disk
and network operations are impossible. Instead, all input is received from the (untrusted) 
machine generating the proof via a special syscall. So if you make a call that looks like a network access,
you might not get a response from `google.com`. Instead, the prover will pick some arbitrary bytes to 
give back to you. The bytes might correspond to an actual response (i.e. if the prover is honest
and made the network request for you) - but they might also be specially crafted to deceive you. 
So, in zero-knowledge mode, great care must be taken to avoid relying on unverified data from the
prover.

In the Sovereign SDK, we try to share code between the `"native"` full node implementation and the
zero-knowledge environment to the greatest extent possible. This minimizes surface area for bugs. 
However, a full node necessarily needs a lot of logic which is unnecessary (and undesirable) to
execute in zero-knowledge. In the SDK, such code is gated behind a `cargo` feature called `"native"`.
This code includes RPC implementations, as well as logic to pre-process some data into formats which
are easier for the zero-knowledge code to verify. 


## The Rollup Interface

If you squint hard enough, a zk-rollup is made of three separate components. There's an underlying
blockchain ("Data Availability layer"), a set of transaction execution rules ("a State Transition Function")
and a zero-knowledge proof system (a "ZKVM" for zero-knowledge virtual machine). In the abstract, it seems 
like it should be possible to take the same transaction processing logic (i.e. the EVM) and deploy it on
top of many different DA layers. Similarly, you _should_ be able to take the same execution logic and compile
it down to several different proof systems - in the same way that you can take the same code an run it on 
Risc0 or SP1. 

Unfortunately, separating these components can be tricky in practice. For example, the OP Stack relies on an 
Ethereum smart contract to enforce its censorship resistance guarantees - so, you can't 
easily take an OP stack rollup and deploy it on a non-EVM chain. 

In the Sovereign SDK, flexibility is a primary design goal. So we take care to codify this separation of concerns
into the framework from the very beginning. With Sovereign, it's possible to run any `State Transition Function`
alongside any `Da Service`































This document provides an overview of the major abstractions offered by the SDK. 

- Rollup Interface (STF + DA service + DA verifier)
- sov-modules (`Runtime`, `Module`, stf-blueprint w/ account abstraction, state abstractions)
- sov-sequencer
- sov-db
- Rockbound
