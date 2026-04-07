---
name: arcium
description: >
  Build privacy-preserving Solana apps with Arcium MPC. Use when writing
  Arcis circuits, Anchor programs with queue_computation, encrypting inputs
  with @arcium-hq/client, or debugging MPC computation failures. Covers
  confidential DeFi, sealed-bid auctions, voting, hidden game state.
license: MIT
compatibility: Bundled mcp.json configures the Arcium MCP server automatically.
metadata:
  author: arcium-hq
  version: "2.0"
---

# Arcium

Encrypted computation on Solana via MPC. Data stays encrypted during computation.

**MCP Tool**: `search_arcium_docs` -- query with natural language for API details, examples, and guides. Pair with `get_page_arcium_docs` to fetch the full page after a search hit.

## When to Use

**Use when:**
- You need trustless computation -- cryptographically guaranteed, no single party sees the data
- Multiple parties compute on combined data without revealing inputs
- On-chain state must remain encrypted but computable
- Privacy: sealed-bid auctions, voting, hidden game state, dark pools, confidential DeFi

**Constraints:**
- Fixed loop bounds required (no variable-length iteration)

## Intent Router

Identify what you're building, then read the linked reference before coding. MCP is fallback for live API details.

| Intent | Read | MCP Fallback |
|--------|------|-------------|
| First Arcium app | [minimal-circuit.md](examples/minimal-circuit.md) | "hello world tutorial" |
| Choose a pattern (stateless, stateful, multi-party) | [patterns.md](examples/patterns.md) | "arcium examples" |
| Circuit syntax (`#[encrypted]`, `#[instruction]`) | [patterns.md](examples/patterns.md) | "arcis encrypted instruction" |
| Shared vs Mxe encryption | See [Encryption Context](#encryption-context) below | "Shared vs Mxe encryption" |
| ArgBuilder ordering / ciphertext errors | [troubleshooting.md -- ArgBuilder Ordering Errors](references/troubleshooting.md#argbuilder-ordering-errors) | "ArgBuilder encrypted plaintext" |
| Callback not firing / computation stuck | [troubleshooting.md -- Computation Never Finalizes](references/troubleshooting.md#computation-never-finalizes) | "arcium_callback queue_computation" |
| Nonce / decryption errors | [troubleshooting.md -- Nonce Errors](references/troubleshooting.md#nonce-errors) | "RescueCipher encrypt nonce" |
| Client-side encryption (RescueCipher, x25519) | [minimal-circuit.md](examples/minimal-circuit.md) -- Test section | "RescueCipher encrypt nonce" |
| Deployment (devnet/mainnet) | MCP primary | "arcium deploy cluster-offset" |
| Version / installation requirements | MCP primary | "arcium installation anchor solana" |

## Core Pattern: Three Functions

Every computation needs three functions in your Solana program:

| Function | Purpose | When Called |
|----------|---------|-------------|
| `init_<name>_comp_def` | Initialize computation definition | Once per instruction |
| `<name>` | Build args + queue computation | Each request |
| `<name>_callback` | Handle result from ARX nodes | After MPC completes |

```rust
const COMP_DEF_OFFSET_FLIP: u32 = comp_def_offset("flip");

// 1. INIT (once per instruction type)
pub fn init_flip_comp_def(ctx: Context<InitFlipCompDef>) -> Result<()> {
    init_comp_def(ctx.accounts, None, None)
}

// 2. QUEUE (each computation)
pub fn flip(ctx: Context<Flip>, offset: u64, ...) -> Result<()> {
    let args = ArgBuilder::new()...build();
    queue_computation(ctx.accounts, offset, args,
        vec![FlipCallback::callback_ix(offset, &ctx.accounts.mxe_account, &[])?],
        1, 0,
    )?;
    Ok(())
}

// 3. CALLBACK (after MPC completes)
#[arcium_callback(encrypted_ix = "flip")]
pub fn flip_callback(ctx: Context<FlipCallback>,
    output: SignedComputationOutputs<FlipOutput>) -> Result<()> {
    let result = output.verify_output(...)?;
    // Use result...
}
```

**Encryption size**: RescueCipher encrypts any scalar to 32 bytes regardless of type.
Formula: `ciphertext_size = 32 * number_of_scalar_values`. See [troubleshooting.md](references/troubleshooting.md) for the full size table.

## Encryption Context

| Scenario | Use |
|----------|-----|
| User inputs, results returned to user | `Enc<Shared, T>` |
| Internal state users shouldn't access | `Enc<Mxe, T>` |
| State persisted across computations | `Enc<Mxe, T>` |
| Final reveal to all parties | `.reveal()` |

## Gotchas

**NEVER:**
- NEVER reuse a nonce — every `cipher.encrypt()` call needs a fresh `randomBytes(16)`
- NEVER combine multiple ciphertexts into one ArgBuilder call — each encrypted scalar is its own `[u8; 32]` call
- NEVER omit `.x25519_pubkey()` for `Enc<Shared, T>` (silent failure); `Enc<Mxe, T>` skips it

### Critical (silent failures)
- **Macro string matching**: All macro strings must exactly match `#[instruction] fn NAME` across `#[arcium_callback]`, `comp_def_offset()`, `#[init_computation_definition_accounts]`, `#[queue_computation_accounts]`, `#[callback_accounts]`
- **ArgBuilder ordering**: Calls must match circuit parameter order left-to-right. For `Enc<Shared, T>`: `.x25519_pubkey()` then `.plaintext_u128(nonce)` then ciphertexts. For `Enc<Mxe, T>`: `.plaintext_u128(nonce)` then ciphertexts. Missing `.x25519_pubkey()` for Shared = silent failure.
- **Division by secret zero**: Guard divisors with the safe divisor pattern -- both branches execute in MPC, so the division always runs. See [patterns.md](examples/patterns.md) pattern #13.

### Warning (wrong results)
- **Nonce reuse**: Same nonce for multiple encryptions = garbled output. Use unique `randomBytes(16)` per encryption.
- **Callback account writability**: Pass extra accounts via `CallbackAccount { pubkey, is_writable: true }` in `callback_ix(..., &[...])`. Also mark `#[account(mut)]` in callback struct. Accounts cannot be created or resized during callbacks.
- **Output struct naming**: Circuit `fn add_together` generates `AddTogetherOutput`. Single returns use `field_0` (a `SharedEncryptedStruct<1>` or `MXEEncryptedStruct<1>` with `.ciphertexts` and `.nonce`). Tuple returns nest `field_0`, `field_1`, etc.

### Tips
- Prefer arithmetic over comparisons (cheaper in MPC)
- Comparisons/divisions are cheaper with narrower types (`u64` vs `u128`); storage cost is identical

## CLI Quick Reference

```bash
arcium init my-project     # Create project
arcium build               # Compile circuits + program
arcium test                # Run with local ARX nodes
arcium deploy \
  --cluster-offset <n> \
  --recovery-set-size <n> \
  --keypair-path <path> \
  --rpc-url <url>          # optional but recommended; MCP: "arcium deploy"
```

## Project Structure

```
my-project/
├── encrypted-ixs/src/lib.rs  # Arcis circuits (MPC code)
├── programs/*/src/lib.rs     # Anchor program
└── tests/*.ts                # TypeScript tests
```

## Verification Checklist

**Circuit:**
- [ ] `arcium build` compiles without errors
- [ ] No `break`/`continue`/`return`/variable-length loops
- [ ] `#[instruction]` fn names are consistent across all macros

**Program:**
- [ ] Every circuit fn has init + invoke + callback instructions
- [ ] `#[arcium_callback(encrypted_ix = "...")]` matches circuit fn name exactly
- [ ] Extra callback accounts passed via `CallbackAccount { pubkey, is_writable: true }` AND `#[account(mut)]` in callback struct

**Client:**
- [ ] Unique nonce per encryption (no reuse across calls)
- [ ] ArgBuilder call order matches circuit fn parameter order left-to-right
- [ ] `.x25519_pubkey()` included for every `Enc<Shared, T>` parameter
- [ ] Cluster offset matches deployment environment

**Deploy:**
- [ ] `arcium test` passes locally before deploy
- [ ] RPC endpoint is reliable (not default Solana RPC)

## Resources

| Resource | URL | Use For |
|----------|-----|---------|
| MCP Server | [docs.arcium.com/mcp](https://docs.arcium.com/mcp) | AI-searchable docs (primary reference) |
| Official Docs | [docs.arcium.com/developers](https://docs.arcium.com/developers/) | Full API reference, guides |
| Examples Repo | [github.com/arcium-hq/examples](https://github.com/arcium-hq/examples) | Production patterns |
| TypeScript SDK | [ts.arcium.com](https://ts.arcium.com/) | @arcium-hq/client API |
| Troubleshooting | [troubleshooting.md](references/troubleshooting.md) | Hard-to-debug errors |
| Patterns | [patterns.md](examples/patterns.md) | Curated code examples |
| Minimal Example | [minimal-circuit.md](examples/minimal-circuit.md) | Complete working app |
