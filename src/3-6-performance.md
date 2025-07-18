# Understanding Performance 

### State Access

The vast majority of the cost of executing a Sovereign SDK transaction comes from state accesses. When calling `item.set(&value)`, the SDK serializes your value and stores the bytes in cache. When time you access a value using `item.get()`, the SDK deserializes a fresh copy of your value from the bytes held in cache, falling back to disk if necessary.

Each time you access a value that's not in cache, the SDK has to generate a merkle proof of the value, which it will consume when it's time to generate a zero-knowledge proof. Similarly, each time you write a new value, the SDK has to generate a merkle update proof. This makes reading/writing to a `hot` value at least an order of magnitude cheaper than writing to a `cold` one (where `hot` means that the value has already been accessed in the current block.) So, if you have state items that are frequently accessed together, it's a good idea to bundle them into a single `StateValue` or store them under the same key in a `StateMap`.

As a rule of thumb, for each 10% locality, you should be willing to add an extra 200 bytes to your `StateValue`. In other words, if two values are accessed together 30% of the time, you should put them together unless either of the state items is bigger than 600 bytes. (Exception: If two items are always accessed together, you should always group them together - no questions asked).

### Cryptography

The other common source of performance woes is heavy-duty cryptography. If you need to do any cryptographic operations, check whether the `Spec` trait provides a method in its `Spec::CryptoSpec` that already does what you want. If it does, use that - the SDK will ensure you get an implementation which is optimized for the SDK's peculiar requirements. If you need access to more exotic cryptography, you can use pretty much any existing Rust library - but be aware that the performance penalty might be severe when it comes time to prove your module's execution, which could limit your total throughput. If you do need advanced cryptography, you may need to pick an implementation that's suited to a particular `ZKVM` (like `SP1` or `Risc0`) and only use that vm with your module.

### Building for Scale

By keeping these performance principles in mind, bundling hot state and using optimized cryptography, you can design your modules to be highly efficient, ensuring your rollup can scale to meet user demand.

While building custom logic is powerful, you don't always have to start from scratch. The Sovereign SDK comes with a rich set of **"Prebuilt Modules"** for common tasks like token management, bridging, and sequencer orchestration. The next section provides an overview of these modules, which you can leverage to accelerate your development.