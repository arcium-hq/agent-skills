# Minimal Arcium App

Complete example: encrypted sum of two numbers.

## Circuit (encrypted-ixs/src/lib.rs)

```rust
use arcis::*;

#[encrypted]
mod circuits {
    use arcis::*;

    pub struct TwoNumbers {
        pub a: u64,
        pub b: u64,
    }

    #[instruction]
    pub fn add(input: Enc<Shared, TwoNumbers>) -> u64 {
        let nums = input.to_arcis();
        (nums.a + nums.b).reveal()
    }
}
```

## Program (programs/adder/src/lib.rs)

```rust
use anchor_lang::prelude::*;
use arcium_anchor::prelude::*;

const COMP_DEF_OFFSET_ADD: u32 = comp_def_offset("add");

declare_id!("...");

#[arcium_program]
pub mod adder {
    use super::*;

    pub fn init_add_comp_def(ctx: Context<InitAddCompDef>) -> Result<()> {
        init_comp_def(ctx.accounts, None, None)
    }

    pub fn add(
        ctx: Context<Add>,
        offset: u64,
        encrypted_a: [u8; 32],  // Encrypted u64 (always 32 bytes from RescueCipher)
        encrypted_b: [u8; 32],  // Encrypted u64 (always 32 bytes from RescueCipher)
        pub_key: [u8; 32],
        nonce: u128,
    ) -> Result<()> {
        let args = ArgBuilder::new()
            .x25519_pubkey(pub_key)
            .plaintext_u128(nonce)
            .encrypted_u64(encrypted_a)
            .encrypted_u64(encrypted_b)
            .build();

        ctx.accounts.sign_pda_account.bump = ctx.bumps.sign_pda_account;

        queue_computation(
            ctx.accounts, offset, args,
            vec![AddCallback::callback_ix(offset, &ctx.accounts.mxe_account, &[])?],
            1, 0,
        )?;
        Ok(())
    }

    #[arcium_callback(encrypted_ix = "add")]
    pub fn add_callback(
        ctx: Context<AddCallback>,
        output: SignedComputationOutputs<AddOutput>,
    ) -> Result<()> {
        let result = output.verify_output(
            &ctx.accounts.cluster_account,
            &ctx.accounts.computation_account,
        )?;
        emit!(AddEvent { sum: result.field_0 });
        Ok(())
    }
}

#[event]
pub struct AddEvent {
    pub sum: u64,
}

#[init_computation_definition_accounts("add", payer)]
#[derive(Accounts)]
pub struct InitAddCompDef<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,
    #[account(mut, address = derive_mxe_pda!())]
    pub mxe_account: Box<Account<'info, MXEAccount>>,
    #[account(mut)]
    /// CHECK: comp_def_account, checked by arcium program.
    pub comp_def_account: UncheckedAccount<'info>,
    #[account(mut, address = derive_mxe_lut_pda!(mxe_account.lut_offset_slot))]
    /// CHECK: address_lookup_table, checked by arcium program.
    pub address_lookup_table: UncheckedAccount<'info>,
    #[account(address = LUT_PROGRAM_ID)]
    /// CHECK: lut_program is the Address Lookup Table program.
    pub lut_program: UncheckedAccount<'info>,
    pub arcium_program: Program<'info, Arcium>,
    pub system_program: Program<'info, System>,
}

#[queue_computation_accounts("add", payer)]
#[derive(Accounts)]
#[instruction(offset: u64)]
pub struct Add<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,
    #[account(
        init_if_needed,
        space = 9,
        payer = payer,
        seeds = [&SIGN_PDA_SEED],
        bump,
        address = derive_sign_pda!(),
    )]
    pub sign_pda_account: Account<'info, ArciumSignerAccount>,
    #[account(address = derive_mxe_pda!())]
    pub mxe_account: Account<'info, MXEAccount>,
    #[account(mut, address = derive_mempool_pda!(mxe_account, ErrorCode::ClusterNotSet))]
    /// CHECK: mempool_account, checked by the arcium program
    pub mempool_account: UncheckedAccount<'info>,
    #[account(mut, address = derive_execpool_pda!(mxe_account, ErrorCode::ClusterNotSet))]
    /// CHECK: executing_pool, checked by the arcium program
    pub executing_pool: UncheckedAccount<'info>,
    #[account(mut, address = derive_comp_pda!(offset, mxe_account, ErrorCode::ClusterNotSet))]
    /// CHECK: computation_account, checked by the arcium program.
    pub computation_account: UncheckedAccount<'info>,
    #[account(address = derive_comp_def_pda!(COMP_DEF_OFFSET_ADD))]
    pub comp_def_account: Account<'info, ComputationDefinitionAccount>,
    #[account(mut, address = derive_cluster_pda!(mxe_account, ErrorCode::ClusterNotSet))]
    pub cluster_account: Account<'info, Cluster>,
    #[account(mut, address = ARCIUM_FEE_POOL_ACCOUNT_ADDRESS)]
    pub pool_account: Account<'info, FeePool>,
    #[account(mut, address = ARCIUM_CLOCK_ACCOUNT_ADDRESS)]
    pub clock_account: Account<'info, ClockAccount>,
    pub system_program: Program<'info, System>,
    pub arcium_program: Program<'info, Arcium>,
}

#[callback_accounts("add")]
#[derive(Accounts)]
pub struct AddCallback<'info> {
    pub arcium_program: Program<'info, Arcium>,
    #[account(address = derive_comp_def_pda!(COMP_DEF_OFFSET_ADD))]
    pub comp_def_account: Account<'info, ComputationDefinitionAccount>,
    #[account(address = derive_mxe_pda!())]
    pub mxe_account: Account<'info, MXEAccount>,
    pub computation_account: UncheckedAccount<'info>,
    #[account(address = derive_cluster_pda!(mxe_account, ErrorCode::ClusterNotSet))]
    pub cluster_account: Account<'info, Cluster>,
    #[account(address = ::anchor_lang::solana_program::sysvar::instructions::ID)]
    pub instructions_sysvar: AccountInfo<'info>,
}

#[error_code]
pub enum ErrorCode {
    #[msg("Cluster not set")]
    ClusterNotSet,
    #[msg("Aborted computation")]
    AbortedComputation,
}
```

## Test (tests/adder.ts)

```typescript
import * as anchor from "@coral-xyz/anchor";
import { randomBytes } from "crypto";
import { sha256 } from "@noble/hashes/sha256";
import {
  getMXEPublicKey, RescueCipher, x25519, deserializeLE,
  awaitComputationFinalization, getArciumEnv, getCompDefAccOffset,
  getMXEAccAddress, getMempoolAccAddress, getCompDefAccAddress,
  getExecutingPoolAccAddress, getComputationAccAddress, getClusterAccAddress,
} from "@arcium-hq/client";

describe("adder", () => {
  const provider = anchor.AnchorProvider.env();
  const program = anchor.workspace.Adder;

  it("adds two encrypted numbers", async () => {
    // Init comp def (once)
    await program.methods.initAddCompDef().rpc();

    // ARX nodes may take time to initialize after MXE account creation
    // Retry with backoff until public key is available
    let mxePublicKey: Uint8Array | null = null;
    for (let i = 0; i < 20; i++) {
      const key = await getMXEPublicKey(provider, program.programId);
      if (key) { mxePublicKey = key; break; }
      await new Promise(r => setTimeout(r, 500));
    }
    if (!mxePublicKey) throw new Error("Failed to get MXE public key after 20 attempts");

    // TESTING: Ephemeral random keys
    const privateKey = x25519.utils.randomSecretKey();
    const publicKey = x25519.getPublicKey(privateKey);

    // Key exchange + encrypt
    const sharedSecret = x25519.getSharedSecret(privateKey, mxePublicKey);
    const cipher = new RescueCipher(sharedSecret);

    const a = BigInt(10);
    const b = BigInt(32);
    const nonce = randomBytes(16);
    const ciphertext = cipher.encrypt([a, b], nonce);

    const offset = new anchor.BN(randomBytes(8), "hex");

    // Submit computation
    await program.methods
      .add(offset, Array.from(ciphertext[0]), Array.from(ciphertext[1]),
           Array.from(publicKey), new anchor.BN(deserializeLE(nonce).toString()))
      .accountsPartial({
        computationAccount: getComputationAccAddress(
          getArciumEnv().arciumClusterOffset, offset),
        clusterAccount: getClusterAccAddress(getArciumEnv().arciumClusterOffset),
        mxeAccount: getMXEAccAddress(program.programId),
        mempoolAccount: getMempoolAccAddress(getArciumEnv().arciumClusterOffset),
        executingPool: getExecutingPoolAccAddress(getArciumEnv().arciumClusterOffset),
        compDefAccount: getCompDefAccAddress(program.programId,
          Buffer.from(getCompDefAccOffset("add")).readUInt32LE()),
      })
      .rpc({ skipPreflight: true, commitment: "confirmed" });

    // Await result
    await awaitComputationFinalization(provider, offset, program.programId);
    // Result: 42
  });
});
```

## Production Key Derivation

For production apps, derive deterministic keys from wallet signatures:

```typescript
// PRODUCTION: Derive keys from wallet signature (recoverable)
async function deriveEncryptionKeys(
  wallet: { signMessage: (msg: Uint8Array) => Promise<Uint8Array> },
  programId: PublicKey
): Promise<{ privateKey: Uint8Array; publicKey: Uint8Array }> {
  const message = `Arcium encryption key for ${programId.toBase58()}`;
  const signature = await wallet.signMessage(new TextEncoder().encode(message));
  const privateKey = sha256(signature).slice(0, 32);
  const publicKey = x25519.getPublicKey(privateKey);
  return { privateKey, publicKey };
}

// Usage
const { privateKey, publicKey } = await deriveEncryptionKeys(wallet, program.programId);
const sharedSecret = x25519.getSharedSecret(privateKey, mxePublicKey);
const cipher = new RescueCipher(sharedSecret);
```
