# Solana Integration

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

## Initialize Computation Definition

One-time setup before computations can be queued:

```rust
pub fn init_flip_comp_def(ctx: Context<InitFlipCompDef>) -> Result<()> {
    init_comp_def(ctx.accounts, None, None)?;
    Ok(())
}

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
| `.encrypted_u64(ct)` | Encrypted u64 ciphertext |
| `.encrypted_bool(ct)` | Encrypted boolean |
| `.account(pubkey, offset, size)` | Read encrypted data from account |

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
