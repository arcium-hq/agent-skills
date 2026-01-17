---
name: arcium
description: Build privacy-preserving Solana apps with Arcium MPC. Use when writing Arcis circuits (#[encrypted], #[instruction]), integrating with Anchor programs (queue_computation, callbacks), or encrypting client inputs (@arcium-hq/client). Covers encrypted state, MPC randomness, and multi-party computation patterns.
---

# Arcium

## When to Use Arcium

**Use when:**
- Multiple parties compute on combined data without revealing inputs
- On-chain state must remain encrypted but computable
- Privacy required: sealed-bid auctions, private voting, hidden game state

**Don't use when:**
- Single-party computation (use standard encryption)
- Sub-second latency required (MPC has network overhead)
- Complex data-dependent control flow (see [mental model](references/mental-model.md))

## Architecture

```
+-------------------------------------------------------------+
|  1. ARCIS CIRCUIT (Rust)                                    |
|     #[encrypted] mod + #[instruction] fn                    |
|     Compiles to MPC circuit executed by ARX nodes           |
+-------------------------------------------------------------+
|  2. SOLANA PROGRAM (Anchor)                                 |
|     #[arcium_program] + queue_computation + callbacks       |
|     Orchestrates computation lifecycle on-chain             |
+-------------------------------------------------------------+
|  3. CLIENT (TypeScript)                                     |
|     @arcium-hq/client + RescueCipher encryption             |
|     Encrypts inputs, submits transactions, awaits results   |
+-------------------------------------------------------------+
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

**Note**: `arcium test` auto-starts local Solana validator and ARX nodes.

## Deep Dive References

- **[Mental Model](references/mental-model.md)** - Why MPC constraints exist
- **[Arcis Circuits](references/arcis-circuits.md)** - `Enc<C,T>`, `.to_arcis()/.from_arcis()`, `ArcisRNG`
- **[Solana Integration](references/solana-program.md)** - Account macros, `ArgBuilder`, callbacks
- **[TypeScript Client](references/typescript-client.md)** - `RescueCipher`, key exchange
- **[Troubleshooting](references/troubleshooting.md)** - Common errors and solutions

## Examples

- **[Minimal Circuit](examples/minimal-circuit.md)** - Complete working Arcium app
- **[Patterns](examples/patterns.md)** - State, randomness, multi-party patterns

## External

- [Arcium Docs](https://docs.arcium.com/developers/)
- [Examples Repo](https://github.com/arcium-hq/examples)
