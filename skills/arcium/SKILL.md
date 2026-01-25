---
name: arcium
description: Build encrypted computation apps on Solana with Arcium MPC. Use when writing Arcis circuits, Anchor programs with queue_computation, or encrypting client inputs with @arcium-hq/client.
---

# Arcium

Encrypted computation on Solana via MPC. Data stays encrypted during computation.

**Use MCP**: Search via the [Arcium MCP](https://docs.arcium.com/mcp) for detailed API docs.

## Key Concepts

- **ARX** = Arcium eXecution nodes (distributed network)
- **MXE** = MPC Execution Environment (runtime)
- **MPC** = Multi-Party Computation (cryptographic protocol)

## When to Use

**Use when:**
- Multiple parties compute on combined data without revealing inputs
- On-chain state must remain encrypted but computable
- Privacy: sealed-bid auctions, voting, hidden game state

**Don't use when:**
- Single-party computation (use standard encryption)
- Sub-second latency required (MPC has network overhead)
- Complex data-dependent control flow (MPC has fixed circuit constraints)

## Core Pattern: Three Functions

Every computation needs three functions in your Solana program:

| Function | Purpose | When Called |
|----------|---------|-------------|
| `init_<name>_comp_def` | Initialize computation definition | Once per instruction |
| `<name>` | Build args + queue computation | Each request |
| `<name>_callback` | Handle result from ARX nodes | After MPC completes |

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

## Decision Nudges

| Need | MCP Search |
|------|------------|
| Encryption types (Shared vs Mxe) | "Shared vs Mxe encryption context" |
| ArgBuilder methods | "ArgBuilder encrypted plaintext" |
| Circuit syntax | "arcis encrypted instruction" |
| Client-side encryption | "RescueCipher encrypt nonce" |
| Callback output types | "SignedComputationOutputs verify_output" |
| Re-encryption / sealing | "re-encrypt recipient sealing" |
| Unsupported types | "arcis unsupported Vec String" |
| Account macros | "queue_computation_accounts callback_accounts" |
| Deployment | "arcium deploy cluster-offset devnet" |
| Version requirements | "arcium installation anchor solana" |

## Encryption Size

RescueCipher encrypts ANY value to 32 bytes:
- u8 -> 32 bytes
- u128 -> 32 bytes
- bool -> 32 bytes

This affects account sizing when storing encrypted data.

## CLI Quick Reference

```bash
arcium init my-project     # Create project
arcium build               # Compile circuits + program
arcium test                # Run with local ARX nodes
arcium deploy --cluster-offset <n>  # See MCP: "arcium deploy" for cluster offsets
```

## Gotchas

### Macro String Matching (Critical)
All macro strings must exactly match `#[instruction] fn NAME`:
```rust
#[arcium_callback(encrypted_ix = "flip")]  // Must match fn flip in circuit
comp_def_offset("flip")                    // Same name
#[queue_computation_accounts("flip", payer)]
#[callback_accounts("flip")]
```

### Output Struct Naming
Circuit `fn add_together` generates `AddTogetherOutput`. Tuple returns use `field_0`, `field_1`, etc.

### Callback Nonce
`SignedComputationOutputs` includes the nonce directly - no manual N+1 calculation needed.

### Early Exit Pattern
No `break`/`continue` allowed. Use done flag:
```rust
let mut done = false;
for i in 0..MAX { if !done && condition { done = true; result = i; } }
```

### Fixed-Size Workarounds
- `([T; MAX], usize)` - array with length counter
- `u8::MAX` as sentinel for "empty"
- `{ value: T, is_some: bool }` - Option replacement

### Optimization Tips
- Batch `.from_arcis()` calls to minimize conversions
- Prefer arithmetic over comparisons (cheaper in MPC)
- Use `u64` over `u128` when possible (fewer bits = faster)

## Project Structure

```
my-project/
├── encrypted-ixs/src/lib.rs  # Arcis circuits (MPC code)
├── programs/*/src/lib.rs     # Anchor program
└── tests/*.ts                # TypeScript tests
```

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
