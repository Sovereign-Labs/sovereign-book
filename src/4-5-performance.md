# Understanding Performance 

The performance of your modules directly impacts your rollup's throughput, latency, and user transaction costs. While the SDK handles many optimizations automatically, your design choices—especially regarding state accesses and cryptography—are the biggest levers you have.

## State Access: The Golden Rule is to Minimize Distinct Accesses

**The Problem: Every State Access Has a High Fixed Cost**

The vast majority of the cost of executing a transaction comes from state accesses. Each time you call `.get()` or `.set()` on a *distinct* state item for the first time in a block (a "cold" access), the SDK must generate a Merkle proof for that item. This proof is required by the ZK-prover to verify that the data is part of the correct state root. Generating these proofs is expensive.

Accessing a value that has already been touched in the current block (a "hot" access) is much cheaper because the proof has already been generated and cached.

**The Solution: Bundle Related Data**

The most effective optimization is to group data that is frequently read or written together into a single `StateValue`.

Consider a user profile module. A naive implementation might look like this:

```rust
// ANTI-PATTERN: Separated state items
#[state]
pub usernames: StateMap<S::Address, String>,
#[state]
pub bios: StateMap<S::Address, String>,
#[state]
pub follower_counts: StateMap<S::Address, u64>,
```

Loading a single user's profile would require three distinct (and expensive) state accesses. A much better approach is to bundle the data:

```rust
// GOOD PATTERN: Bundled state
pub struct ProfileData {
    pub username: String,
    pub bio: String,
    pub follower_count: u64,
}

#[state]
pub profiles: StateMap<S::Address, ProfileData>,
```

Now, loading a profile requires only one state access.

**The Trade-off:** This bundling increases the size of the value being read from storage, but it saves the massive fixed cost of generating a new Merkle proof for a separate state access. This trade-off is almost always worth it.

While precise numbers can change with SDK updates, the cost of reading even a few hundred extra bytes is negligible compared to the cost of a distinct "cold" state access. Therefore, the guiding principle is: if data items have a reasonable probability of being used together, you should bundle them. If two items are **always** accessed together, they should **always** be in the same state item, regardless of size.

## Cryptography: Use ZK-Optimized Implementations

**The Problem: General-Purpose Crypto is Slow to Prove**

The other common source of performance issues is heavy-duty cryptography. Many standard Rust crypto libraries are not optimized for ZK environments and can be extremely slow to prove, creating a bottleneck that limits your rollup's throughput.

**The Solution Hierarchy:**

1.  **Preferred:** Use the implementations provided by the `Spec::CryptoSpec` associated type. These are guaranteed to be selected for their ZK-friendly performance.
2.  **If you must use an external library:** Be aware of the potential for a severe performance penalty during proof generation.
3.  **For advanced, specialized needs:** Consider using a library tailored to a specific ZKVM (like `SP1` or `Risc0`). This will give you better performance but will tie your module to that specific proving system.

## Next up: Prebuilt Modules

Building custom modules is powerful, but you don't always have to start from scratch. The next chapter introduces the SDK's **"Prebuilt Modules,"** which provide ready-to-use solutions for common tasks like token management and bridging.