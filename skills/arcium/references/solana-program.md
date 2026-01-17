# Solana Integration

## Macro String Matching (Critical)

All macro string arguments must **exactly match** the `#[instruction]` function name in your circuit:

| Macro | String Argument | Must Match |
|-------|-----------------|------------|
| `#[arcium_callback(encrypted_ix = "X")]` | `"X"` | `#[instruction] fn X` in circuit |
| `#[init_computation_definition_accounts("X", payer)]` | `"X"` | Same `X` |
| `#[queue_computation_accounts("X", payer)]` | `"X"` | Same `X` |
| `#[callback_accounts("X")]` | `"X"` | Same `X` |
| `comp_def_offset("X")` | `"X"` | Same `X` |

**Example**: If circuit has `#[instruction] pub fn flip(...)`, then ALL macros must use `"flip"`.

```rust
// Circuit (encrypted-ixs/src/lib.rs)
#[instruction]
pub fn flip(input: Enc<Shared, UserChoice>) -> bool { ... }

// Program - all must match "flip"
const COMP_DEF_OFFSET_FLIP: u32 = comp_def_offset("flip");

#[init_computation_definition_accounts("flip", payer)]
pub struct InitFlipCompDef<'info> { ... }

#[queue_computation_accounts("flip", payer)]
pub struct Flip<'info> { ... }

#[arcium_callback(encrypted_ix = "flip")]
pub fn flip_callback(...) { ... }

#[callback_accounts("flip")]
pub struct FlipCallback<'info> { ... }
```

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Circuit function | snake_case | `fn flip`, `fn player_move` |
| Output struct | PascalCase + "Output" | `FlipOutput` |
| Tuple fields | `field_0`, `field_1`... | Anchor limitation |
| Callback function | snake_case + `_callback` | `flip_callback` |
| Init function | `init_<name>_comp_def` | `init_flip_comp_def` |
| Offset constant | `COMP_DEF_OFFSET_<NAME>` | `COMP_DEF_OFFSET_FLIP` |
| Event struct | PascalCase + "Event" | `FlipEvent` |
| Account struct | PascalCase by instruction | `Flip`, `FlipCallback` |

## Program Setup

```rust
use anchor_lang::prelude::*;
use arcium_anchor::prelude::*;

const COMP_DEF_OFFSET_FLIP: u32 = comp_def_offset("flip");

declare_id!("YourProgramId...");

#[arcium_program]
pub mod my_program {
    use super::*;
    // ...
}
```

**Cargo.toml:**
```toml
[dependencies]
anchor-lang = { version = "0.32.1", features = ["init-if-needed"] }
arcium-client = { version = "0.6.2", default-features = false }
arcium-macros = "0.6.2"
arcium-anchor = "0.6.2"
```

## Three-Function Pattern

Every Arcium instruction requires three functions:

1. **`init_<name>_comp_def`** - Initialize computation definition (once per instruction)
2. **`<name>`** - Build args and queue computation
3. **`<name>_callback`** - Handle result after MPC execution

## Initialize Computation Definition

One-time setup before computations can be queued:

```rust
pub fn init_flip_comp_def(ctx: Context<InitFlipCompDef>) -> Result<()> {
    init_comp_def(ctx.accounts, None, None)?;
    Ok(())
}
```

### Off-Chain Circuit Storage

Circuits can be several MB, making on-chain initialization expensive. Use off-chain storage for large circuits:

```rust
use arcium_client::idl::arcium::types::{CircuitSource, OffChainCircuitSource};
use arcium_macros::circuit_hash;

pub fn init_large_comp_def(ctx: Context<InitLargeCompDef>) -> Result<()> {
    init_comp_def(
        ctx.accounts,
        Some(CircuitSource::OffChain(OffChainCircuitSource {
            source: "https://your-storage.com/circuits/large_circuit.arcis".to_string(),
            hash: circuit_hash!("large_circuit"),  // SHA-256 computed at compile time
        })),
        None,
    )?;
    Ok(())
}
```

**CRITICAL**: You **must** use the `circuit_hash!` macro. Using a placeholder like `[0u8; 32]` causes "Hash verification failed" errors at runtime.

```rust
// WRONG - will fail at runtime
hash: [0u8; 32],

// CORRECT - computes SHA-256 at compile time
hash: circuit_hash!("instruction_name"),
```

**When to use off-chain storage:**
- Circuit file > 100KB
- Multiple instructions sharing similar logic
- Frequently updated circuits (update URL, not on-chain)

```rust
#[init_computation_definition_accounts("flip", payer)]
#[derive(Accounts)]
pub struct InitFlipCompDef<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,
    #[account(mut, address = derive_mxe_pda!())]
    pub mxe_account: Box<Account<'info, MXEAccount>>,
    #[account(mut)]
    pub comp_def_account: UncheckedAccount<'info>,
    pub arcium_program: Program<'info, Arcium>,
    pub system_program: Program<'info, System>,
}
```

## Queue Computation

```rust
pub fn flip(
    ctx: Context<Flip>,
    computation_offset: u64,
    user_choice: [u8; 32],
    pub_key: [u8; 32],
    nonce: u128,
) -> Result<()> {
    let args = ArgBuilder::new()
        .x25519_pubkey(pub_key)
        .plaintext_u128(nonce)
        .encrypted_u8(user_choice)
        .build();

    queue_computation(
        ctx.accounts,
        computation_offset,
        args,
        None,  // Optional additional data
        vec![FlipCallback::callback_ix(
            computation_offset,
            &ctx.accounts.mxe_account,
            &[],  // Additional callback accounts
        )?],
        1,     // Gas units
        0,     // Priority
    )?;
    Ok(())
}
```

### ArgBuilder Methods

| Method | Purpose |
|--------|---------|
| `.x25519_pubkey(key)` | Client's public key for encryption |
| `.plaintext_u128(val)` | Unencrypted value (nonce) |
| `.encrypted_u8(ct)` | Encrypted u8 ciphertext |
| `.encrypted_u16(ct)` | Encrypted u16 ciphertext |
| `.encrypted_u32(ct)` | Encrypted u32 ciphertext |
| `.encrypted_u64(ct)` | Encrypted u64 ciphertext |
| `.encrypted_u128(ct)` | Encrypted u128 ciphertext |
| `.encrypted_bool(ct)` | Encrypted boolean |
| `.account(pubkey, offset, size)` | Read encrypted data from account |

### Account Offset Calculation (Critical)

When using `.account(pubkey, offset, size)` to read encrypted data from an account, you **must calculate the byte offset correctly**. Off-by-one errors cause silent failures or garbage data.

**Byte Sizes:**
| Type | Size |
|------|------|
| Discriminator | 8 bytes |
| `bump: u8` | 1 byte |
| `bool` | 1 byte |
| `u8` | 1 byte |
| `u16` | 2 bytes |
| `u32` | 4 bytes |
| `u64` | 8 bytes |
| `u128` | 16 bytes |
| `Pubkey` | 32 bytes |
| `[u8; N]` | N bytes |

**Example - Voting App:**
```rust
#[account]
pub struct PollState {
    pub bump: u8,              // +1 (offset: 8)
    pub authority: Pubkey,     // +32 (offset: 9)
    pub encrypted_votes: [u8; 64], // Target field (offset: 41)
}

// Offset = discriminator(8) + bump(1) + authority(32) = 41
const ENCRYPTED_VOTES_OFFSET: u32 = 8 + 1 + 32;

let args = ArgBuilder::new()
    // ...
    .account(ctx.accounts.poll_state.key(), ENCRYPTED_VOTES_OFFSET, 64)
    .build();
```

**Example - Sealed Bid Auction:**
```rust
#[account]
pub struct AuctionState {
    pub bump: u8,                  // +1 (offset: 8)
    pub authority: Pubkey,         // +32 (offset: 9)
    pub auction_type: u8,          // +1 (offset: 41)
    pub status: u8,                // +1 (offset: 42)
    pub min_bid: u64,              // +8 (offset: 43)
    pub end_time: u64,             // +8 (offset: 51)
    pub bid_count: u8,             // +1 (offset: 59)
    pub state_nonce: u128,         // +16 (offset: 60)
    pub encrypted_state: [u8; 128], // Target (offset: 76)
}

// Offset = 8 + 1 + 32 + 1 + 1 + 8 + 8 + 1 + 16 = 76
const ENCRYPTED_STATE_OFFSET: u32 = 76;
```

**GOTCHA**: Always count bytes starting from byte 0. The discriminator (8 bytes) is automatically added by Anchor for `#[account]` structs - include it in your offset calculation.

### Ciphertext Size

Each encrypted value produces 32 bytes. For multi-value structs, use multiple ArgBuilder calls. See [Ciphertext Size Mismatch](troubleshooting.md#ciphertext-size-mismatch) for the complete size mapping table.

## Handle Callback

```rust
#[arcium_callback(encrypted_ix = "flip")]
pub fn flip_callback(
    ctx: Context<FlipCallback>,
    output: SignedComputationOutputs<FlipOutput>,
) -> Result<()> {
    let result = output.verify_output(
        &ctx.accounts.cluster_account,
        &ctx.accounts.computation_account,
    )?;

    // Generated types use field_0, field_1 for tuple returns
    emit!(FlipEvent { result: result.field_0 });
    Ok(())
}
```

### The callback_ix() Helper

The `#[callback_accounts]` macro generates a `callback_ix()` method on your callback struct. **Always use this instead of manually building `CallbackInstruction`.**

```rust
// Generated by #[callback_accounts("flip")]
// Auto-includes: arcium_program, comp_def_account, mxe_account,
// computation_account, cluster_account, instructions_sysvar
FlipCallback::callback_ix(
    computation_offset,
    &ctx.accounts.mxe_account,
    &[/* additional CallbackAccounts */],
)?
```

**With custom accounts:**
```rust
vec![FlipCallback::callback_ix(
    offset,
    &ctx.accounts.mxe_account,
    &[
        CallbackAccount::new(ctx.accounts.game_state.key(), true, false),
        CallbackAccount::new(ctx.accounts.player.key(), false, false),
    ],
)?]
```

**CallbackAccount parameters:**
- `key: Pubkey` - Account public key
- `is_writable: bool` - Can callback write to this account?
- `is_signer: bool` - Must this account sign? (usually `false` in callbacks)

### Callback Output Struct Generation

The `#[arcium_callback]` macro generates output structs automatically based on your circuit's return type:

**Naming Convention**: Circuit function `fn_name` generates `FnNameOutput` (PascalCase + "Output")

| Circuit Return | Generated Struct | Field Names |
|----------------|------------------|-------------|
| `bool` | `FlipOutput { field_0: bool }` | `field_0` |
| `u64` | `AddOutput { field_0: u64 }` | `field_0` |
| `(u64, bool)` | `MyOutput { field_0: u64, field_1: bool }` | `field_0`, `field_1` |
| `MyStruct { a: u64 }` | `MyOutput { field_0: u64 }` | Flattened to `field_0`, etc. |

**For encrypted outputs** (`Enc<Mxe, T>` or `Enc<Shared, T>`), use:
- `MXEEncryptedStruct<LEN>` for MXE-encrypted outputs
- `SharedEncryptedStruct<LEN>` for client-encrypted outputs

Where `LEN` = number of scalar values in the struct:

| Type | LEN |
|------|-----|
| `u32`, `bool`, single scalar | 1 |
| `(u32, bool)`, two scalars | 2 |
| `[u32; 5]`, array of 5 | 5 |
| `{ a: u64, b: u64 }` | 2 |

These structs contain: `encryption_key: [u8; 32]`, `nonce: [u8; 16]`, `ciphertexts: [[u8; 32]; LEN]`

#[callback_accounts("flip")]
#[derive(Accounts)]
pub struct FlipCallback<'info> {
    pub arcium_program: Program<'info, Arcium>,
    #[account(address = derive_comp_def_pda!(COMP_DEF_OFFSET_FLIP))]
    pub comp_def_account: Account<'info, ComputationDefinitionAccount>,
    #[account(address = derive_mxe_pda!())]
    pub mxe_account: Account<'info, MXEAccount>,
    pub computation_account: UncheckedAccount<'info>,
    #[account(address = derive_cluster_pda!(mxe_account, ErrorCode::ClusterNotSet))]
    pub cluster_account: Account<'info, Cluster>,
    #[account(address = ::anchor_lang::solana_program::sysvar::instructions::ID)]
    pub instructions_sysvar: AccountInfo<'info>,
}
```

## Queue Computation Accounts

**Note**: Use `Box<Account<>>` for large account types to avoid Solana's 4KB stack limit. The queue computation struct has many accounts, so Box moves them to the heap.

**Rule of thumb**: Use `Box<Account<>>` for queue accounts (many accounts = stack overflow risk). Callback accounts can often use plain `Account<>` (fewer accounts).

```rust
#[queue_computation_accounts("flip", payer)]
#[derive(Accounts)]
pub struct Flip<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,
    #[account(address = derive_mxe_pda!())]
    pub mxe_account: Box<Account<'info, MXEAccount>>,
    #[account(mut, address = derive_mempool_pda!(mxe_account, ErrorCode::MempoolNotSet))]
    pub mempool_account: Box<Account<'info, Mempool>>,
    #[account(mut, address = derive_execpool_pda!(mxe_account, ErrorCode::ExecutingPoolNotSet))]
    pub executing_pool: Box<Account<'info, ExecutingPool>>,
    #[account(address = derive_comp_def_pda!(COMP_DEF_OFFSET_FLIP))]
    pub comp_def_account: Box<Account<'info, ComputationDefinitionAccount>>,
    #[account(mut)]
    pub computation_account: UncheckedAccount<'info>,
    #[account(address = derive_cluster_pda!(mxe_account, ErrorCode::ClusterNotSet))]
    pub cluster_account: Box<Account<'info, Cluster>>,
    pub arcium_program: Program<'info, Arcium>,
    pub system_program: Program<'info, System>,
}
```

## PDA Derivation Macros

| Macro | Purpose |
|-------|---------|
| `derive_mxe_pda!()` | MXE account address |
| `derive_cluster_pda!(mxe, error)` | Cluster account |
| `derive_mempool_pda!(mxe, error)` | Mempool account |
| `derive_execpool_pda!(mxe, error)` | Execution pool |
| `derive_comp_pda!(offset, mxe, error)` | Computation account |
| `derive_comp_def_pda!(offset)` | Computation definition |
| `derive_sign_pda!()` | Signer PDA |

**Note**: `ErrorCode::ClusterNotSet`, `ErrorCode::MempoolNotSet`, etc. are **custom error codes you must define** in your program:

```rust
#[error_code]
pub enum ErrorCode {
    #[msg("Cluster not set")]
    ClusterNotSet,
    #[msg("Mempool not set")]
    MempoolNotSet,
    #[msg("Executing pool not set")]
    ExecutingPoolNotSet,
}
```

### PDA Offset Explained

**`comp_def_offset("instruction_name")`** generates a deterministic u32:
- Internally: `sha256("instruction_name").slice(0, 4)` as little-endian u32
- This offset is **per-instruction**, not per-computation
- Used to derive the computation definition PDA

**`computation_offset`** (u64) is **per-computation**:
- Generated client-side from 8 random bytes
- Used to derive the computation account PDA
- Must be unique for each computation request

```typescript
// Client-side computation offset generation
const computationOffset = new anchor.BN(randomBytes(8), "hex");
```

## Storing Encrypted State

To store MXE-encrypted state on-chain:

```rust
#[account]
pub struct GameState {
    pub encrypted_data: [u8; 64],  // Size depends on your struct
}

// In callback, store the encrypted output
let encrypted_bytes = output.get_encrypted_output(0)?;
ctx.accounts.game_state.encrypted_data = encrypted_bytes;
```

### Storage Pattern Constraints

**Critical**: ARX nodes execute callbacks, so they can't pay for account creation.

| Action | Queue Function | Callback |
|--------|---------------|----------|
| Create account | ✅ (payer funds) | ❌ (nodes can't pay) |
| Resize account | ✅ | ❌ |
| Write to account | ✅ | ✅ (must be writable) |
| Close account | ✅ | ❌ |

**Rules:**
1. **Accounts must exist before callback** - create in queue function
2. **Initialize in queue function, update in callback**:
   ```rust
   // In queue function (flip)
   #[account(init_if_needed, payer = payer, space = 8 + GameState::SIZE)]
   pub game_state: Account<'info, GameState>,

   // In callback (flip_callback)
   #[account(mut)]  // Just mut, not init
   pub game_state: Account<'info, GameState>,
   ```
3. **Writability requires BOTH places**:
   - `CallbackAccount::new(key, true /* is_writable */, false)`
   - `#[account(mut)]` in callback accounts struct
4. **Allocate maximum space upfront** - cannot resize during callback

```rust
// Include custom accounts in callback instruction
vec![FlipCallback::callback_ix(
    offset,
    &ctx.accounts.mxe_account,
    &[CallbackAccount::new(
        ctx.accounts.game_state.key(),
        true,  // is_writable
        false, // is_signer
    )],
)?]
```

## See Also

- [Arcis Circuits](arcis-circuits.md) - Circuit syntax and `Enc<C,T>` types
- [TypeScript Client](typescript-client.md) - Client-side encryption and submission
- [Troubleshooting](troubleshooting.md) - Common errors and solutions
- [Mental Model](mental-model.md) - Why MPC constraints exist
