# Running the Starter Rollup

This chapter is about one thing: **getting to run your first rollup.** We'll do this by cloning a pre-built starter rollup and running it on your local machine. We'll save the fun part—writing your own code—for the next chapter.

## Prerequisites

Before you begin, ensure you have the following installed on your system:

-   **Rust**: Version 1.88 or later. We recommend installing it via [rustup](https://rustup.rs/). The starter repository uses a `rust-toolchain.toml` file to automatically select the correct toolchain version.
-   **Node.js and npm**: Version 20.0 or later. We'll use this for the Typescript client in a later chapter. [Install here.](https://nodejs.org/en/download)
-   **Git**: For cloning the starter repository.

## Running the Rollup

With the prerequisites installed, running the rollup takes just two commands.

1.  **Clone the starter repository:**

    ```bash
    git clone https://github.com/Sovereign-Labs/sov-rollup-starter-wip.git
    cd sov-rollup-starter-wip
    ```

2.  **Build and run the node:**

    ```bash
    cargo run 
    ```

    You should see a stream of log messages, indicating that the rollup node is running and producing new blocks. Keep this terminal window open.

    > **Note:** The first build can take several minutes as Cargo downloads and compiles all the dependencies. Subsequent builds will be much faster.

## Verifying the Node is Running

Open a new terminal window. We can verify that the node is running and all its core components have loaded by querying its list of **modules**.

Modules are the individual building blocks of a Sovereign SDK rollup, each handling a specific feature like token management (`bank`) or the sequencer registry. Let's query the `/modules` endpoint to see which ones are active in the starter rollup:

```bash
curl 'http://127.0.0.1:12346/modules'
```

If everything is working, you should see a JSON response listing the default modules included in the starter rollup, like `bank`, `accounts`, and `sequencer_registry`.

```json
{
  "data": {
    "modules": [
      "bank",
      "sequencer_registry",
      "accounts",
      "value_setter"
      // ... and others
    ]
  },
  "meta": {}
}
```

## What's Next?

Now that you've successfully run the starter rollup, let's get you building your own.