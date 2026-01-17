---
name: arcium
description: Build encrypted computation apps on Solana with Arcium. Use when writing Arcis circuits (#[encrypted], #[instruction]), integrating with Anchor programs (queue_computation, callbacks), or encrypting client inputs (@arcium-hq/client). Covers privacy-preserving computation, confidential state, and computing on encrypted data.
---

# Arcium

Arcium enables **encrypted computation** (also called confidential computing or privacy-preserving computation) on Solana. Data remains encrypted while being computed on by a distributed network of ARX (Arcium eXecution) nodes using MPC (multi-party computation) protocols.

**Key Terms:**
- **ARX** = Arcium eXecution nodes (the distributed network)
- **MXE** = MPC Execution Environment (the runtime that executes encrypted computations)
- **MPC** = Multi-Party Computation (cryptographic protocol enabling computation on secret-shared data)

## Prerequisites

- [Rust](https://www.rust-lang.org/tools/install)
- [Solana CLI 2.3.0](https://docs.solana.com/cli/install-solana-cli-tools) + `solana-keygen new`
- [Yarn](https://yarnpkg.com/getting-started/install)
- [Anchor 0.32.1](https://www.anchor-lang.com/docs/installation)
- [Docker & Docker Compose](https://docs.docker.com/engine/install/)

## Terminology

| Term | Type | Scope | Description |
|------|------|-------|-------------|
| `comp_def_offset` | u32 | Per instruction | `comp_def_offset("flip")` - derived from instruction name hash |
| `computation_offset` | u64 | Per computation | Random bytes - unique identifier for each computation request |
| `cluster_offset` | u32 | Per network | From `arcium list-clusters` - identifies target ARX node cluster |

## Installation

**Quick Install (Mac/Linux - Recommended)**:
```bash
curl --proto '=https' --tlsv1.2 -sSfL https://install.arcium.com/ | bash
```

**Manual Install**:
```bash
# 1. Set TARGET for your platform:
#    - aarch64_macos  (Apple Silicon Mac)
#    - x86_64_macos   (Intel Mac)
#    - aarch64_linux  (ARM64 Linux)
#    - x86_64_linux   (x86-64 Linux)
export TARGET=aarch64_macos  # Change to your platform

# 2. Download and install arcup
curl "https://bin.arcium.com/download/arcup_${TARGET}_0.6.2" -o ~/.cargo/bin/arcup
chmod +x ~/.cargo/bin/arcup

# 3. Install CLI and ARX node
arcup install

# 4. Verify
arcium --version
```

**Note**: Windows not supported. Use WSL2 with Ubuntu.

## Version Requirements

| Component | Version |
|-----------|---------|
| Arcium CLI | 0.6.2 |
| Anchor | 0.32.1 |
| Solana CLI | 2.3.0 |
| Docker Compose | Required |

## When to Use Arcium

**Use when:**
- Multiple parties compute on combined data without revealing inputs
- On-chain state must remain encrypted but computable
- Privacy required: sealed-bid auctions, private voting, hidden game state

**Don't use when:**
- Single-party computation (use standard encryption)
- Sub-second latency required (MPC has network overhead)
- Complex data-dependent control flow (see [mental model](references/mental-model.md))

## Three-Function Pattern

Every Arcium instruction requires **three functions** in your Solana program:

| Function | Purpose | When Called |
|----------|---------|-------------|
| `init_<name>_comp_def` | Initialize computation definition | Once per instruction (setup) |
| `<name>` | Build args + queue computation | Each computation request |
| `<name>_callback` | Handle result from ARX nodes | After MPC execution completes |

```rust
// 1. INIT (once per instruction type)
pub fn init_flip_comp_def(ctx: Context<InitFlipCompDef>) -> Result<()> {
    init_comp_def(ctx.accounts, None, None)
}

// 2. QUEUE (each computation)
pub fn flip(ctx: Context<Flip>, offset: u64, ...) -> Result<()> {
    let args = ArgBuilder::new()...build();
    queue_computation(ctx.accounts, offset, args, None,
        vec![FlipCallback::callback_ix(...)], 1, 0)
}

// 3. CALLBACK (after MPC completes)
#[arcium_callback(encrypted_ix = "flip")]
pub fn flip_callback(ctx: Context<FlipCallback>,
    output: SignedComputationOutputs<FlipOutput>) -> Result<()> {
    let result = output.verify_output(...)?;
    // Use result...
}
```

## Essential Patterns

### Circuit (encrypted-ixs/src/lib.rs)

```rust
use arcis::*;

#[encrypted]
mod circuits {
    use arcis::*;

    pub struct UserChoice { pub choice: bool }

    #[instruction]
    pub fn flip(input: Enc<Shared, UserChoice>) -> bool {
        let input = input.to_arcis();
        let toss = ArcisRNG::bool();
        (input.choice == toss).reveal()
    }
}
```

### Program (programs/*/src/lib.rs)

```rust
#[arcium_program]
pub mod coinflip {
    const COMP_DEF_OFFSET_FLIP: u32 = comp_def_offset("flip");

    pub fn flip(ctx: Context<Flip>, offset: u64, choice: [u8; 32],
                pub_key: [u8; 32], nonce: u128) -> Result<()> {
        let args = ArgBuilder::new()
            .x25519_pubkey(pub_key)
            .plaintext_u128(nonce)
            .encrypted_u8(choice)
            .build();
        queue_computation(ctx.accounts, offset, args, None,
            vec![FlipCallback::callback_ix(offset, &ctx.accounts.mxe_account, &[])?],
            1, 0)
    }

    #[arcium_callback(encrypted_ix = "flip")]
    pub fn flip_callback(ctx: Context<FlipCallback>,
                         output: SignedComputationOutputs<FlipOutput>) -> Result<()> {
        let result = output.verify_output(&ctx.accounts.cluster_account,
                                          &ctx.accounts.computation_account)?;
        emit!(FlipEvent { result: result.field_0 });
        Ok(())
    }
}
```

### Client (tests/*.ts)

```typescript
import { getMXEPublicKey, awaitComputationFinalization,
         RescueCipher, x25519 } from "@arcium-hq/client";

const mxePublicKey = await getMXEPublicKey(provider, programId);
const privateKey = x25519.utils.randomSecretKey();
const publicKey = x25519.getPublicKey(privateKey);
const sharedSecret = x25519.getSharedSecret(privateKey, mxePublicKey);

const cipher = new RescueCipher(sharedSecret);
const nonce = randomBytes(16);
const ciphertext = cipher.encrypt([BigInt(choice)], nonce);

await program.methods.flip(offset, Array.from(ciphertext[0]),
    Array.from(publicKey), deserializeLE(nonce)).rpc();

await awaitComputationFinalization(provider, offset, programId);
```

## CLI Workflow

```bash
arcium init my-project     # Create project structure
cd my-project
arcium build               # Compile circuits + Solana program
arcium test                # Auto-starts localnet, runs tests, cleans up
arcium deploy --cluster-offset <n> --keypair-path <path> --rpc-url <url>
```

**Note**: `arcium test` auto-starts local Solana validator and ARX nodes. Run `arcium --help` for full CLI reference.

### Project Structure

`arcium init` creates:

```
my-project/
├── Anchor.toml
├── Arcium.toml           # Arcium-specific config
├── Cargo.toml
├── encrypted-ixs/        # Arcis circuits (MPC code)
│   └── src/lib.rs
├── programs/             # Anchor programs (on-chain)
│   └── my-project/
│       └── src/lib.rs
├── tests/                # TypeScript tests
└── migrations/
```

### Arcium.toml Configuration

```toml
[localnet]
nodes = 2                     # Minimum 2 ARX nodes required
localnet_timeout_secs = 60    # RPC wait timeout for node startup

[clusters.devnet]
offset = 123                  # Cluster offset for devnet deployment

[clusters.mainnet]
offset = 456                  # Cluster offset for mainnet deployment
```

**Key settings:**
- `nodes`: Number of local ARX nodes (min 2 for MPC)
- `localnet_timeout_secs`: How long to wait for nodes to start
- `clusters.<network>.offset`: Target cluster for deployment (get from `arcium list-clusters`)

### Deployment Cluster Offset

When deploying to devnet, specify a cluster offset to target an ARX node cluster:

```bash
arcium deploy --cluster-offset 123 --keypair-path ~/.config/solana/id.json --rpc-url <devnet-url>
```

## Deployment Checklist

### What is a Cluster?

A **cluster** is a group of ARX nodes that execute your encrypted computations. Each cluster has an **offset** (numeric ID) used to reference it.

| Environment | Cluster Management |
|-------------|-------------------|
| Localnet | Auto-created by `arcium test`, offset managed automatically |
| Devnet | Use `arcium list-clusters` to find available clusters |
| Mainnet | Production clusters (contact Arcium for access) |

### Deployment Steps

1. **Build** circuits + Solana program:
   ```bash
   arcium build
   ```

2. **Get cluster offset** for target network:
   ```bash
   arcium list-clusters  # Note the offset you want
   ```

3. **Deploy** to target cluster:
   ```bash
   arcium deploy \
     --cluster-offset <offset> \
     --keypair-path ~/.config/solana/id.json \
     --rpc-url <network-rpc>
   ```

4. **Initialize** computation definitions (call once per instruction):
   ```typescript
   await program.methods.initFlipCompDef().rpc();
   await program.methods.initVoteCompDef().rpc();
   // ... one init per #[instruction]
   ```

5. **Verify** deployment:
   ```bash
   arcium mxe-info <program-id>  # Shows MXE account details
   ```

### Common Deployment Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| "Cluster not found" | Wrong offset | Run `arcium list-clusters` for valid offsets |
| "Insufficient funds" | Payer wallet empty | Airdrop SOL on devnet: `solana airdrop 2` |
| Init fails | Already initialized | Skip init OR use fresh program ID |
| "MXE account not found" | Deploy not complete | Re-run `arcium deploy` |

### Devnet vs Mainnet

| Aspect | Devnet | Mainnet |
|--------|--------|---------|
| SOL | Free (airdrop) | Real funds required |
| Clusters | Test clusters available | Contact Arcium |
| Reliability | May reset | Production SLA |
| Use for | Testing, development | Production apps |

## Resources

**References:**
- [Mental Model](references/mental-model.md) - Why MPC constraints exist
- [Arcis Circuits](references/arcis-circuits.md) - Circuit syntax, types, RNG
- [Solana Integration](references/solana-program.md) - Account macros, callbacks
- [TypeScript Client](references/typescript-client.md) - Encryption patterns
- [Troubleshooting](references/troubleshooting.md) - Common errors

**Examples:**
- [Minimal Circuit](examples/minimal-circuit.md) - Complete working app
- [Patterns](examples/patterns.md) - State, randomness, multi-party

**External:**
- [Arcium Docs](https://docs.arcium.com/developers/)
- [Examples Repo](https://github.com/arcium-hq/examples)
- [Anchor Docs](https://www.anchor-lang.com/)
