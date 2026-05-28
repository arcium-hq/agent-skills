# Troubleshooting

> Hard-to-debug errors and solutions. For API reference, use MCP.

## Sections
- Common Errors (quick-reference table)
- MXE Public Key is Null
- Computation Never Finalizes
- Circuit Compilation Errors
- Account Derivation Issues
- ArgBuilder Ordering Errors
- Ciphertext Size Mismatch
- Nonce Errors
- Localnet Issues
- CLI Errors
- Common Gotchas
  - Account Offset Calculation Wrong
  - Callback Account Not Written
  - Circuit Hash Verification Failed
  - Game State Invalid
  - Encrypted Output Wrong Size
- Testing Patterns
  - Unit Test Circuit Logic
  - Integration Test with Localnet
  - Test State Transitions
  - Debugging Failed Computations
  - Test Encrypted State Updates

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| "Failed to fetch MXE public key" | Arx nodes not ready | Add retry with exponential backoff |
| Computation never finalizes | Missing/incorrect callback | Verify `callback_ix` registration |
| Type mismatch in circuit | `Enc` owner inconsistency | Match `Shared`/`Mxe` throughout |
| "Stack height exceeded" | Too many CPIs (Cross-Program Invocations) | Reduce instruction nesting |
| "Account not initialized" | Missing `init_computation_def` call | Call initialization first (helper was `init_comp_def` in v0.9, renamed in v0.10) |
| Stack overflow on queue (v0.10) | `MXEAccount` / `Cluster` / `ComputationDefinitionAccount` not boxed | Wrap in `Box<Account<'info, …>>` in `#[queue_computation_accounts]` |
| Callback `verify_output` returned `Err` | Failed computation, BLS verification failure, or output deserialization failure | Log the error and return `AbortedComputation`; inspect Arcium failure events for structured `ExecutionFailure` details |
| "Invalid cluster" | Wrong network | Verify cluster offset |
| "Decryption failed" / garbled output | Nonce reuse or mismatch | Use unique nonce per encryption |
| Ciphertext size mismatch | Wrong array size | Each value = 32 bytes (see table below) |
| Garbage result from division | Secret divisor was zero (UB in MPC) | Guard with `if divisor != 0` pattern |
| Callback transaction too large | Output exceeds ~1232 bytes | Reduce output fields, use EncData/Pack, or split computation |
| `.filter()` compile error | Variable-length output unsupported | Manual loop with fixed-size array + count |
| `<<` compile error | Left shift unsupported in Arcis | Multiply by powers of 2 |
| Float result wrong/clamped | Computed value outside `[-2^75, 2^75)` | Validate range or use integers |
| Enum in circuit fails | Enums unsupported in Arcis | `u8` constants + validation |
| Recursion compile error | Fixed circuit structure required | Iterative loop with fixed bounds |

## MXE Public Key is Null

```typescript
// Wrong - no retry
const key = await getMXEPublicKey(provider, programId);

// Right - with retry
async function getMXEPublicKeyWithRetry(provider, programId, maxRetries = 20) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    const key = await getMXEPublicKey(provider, programId);
    if (key) return key;
    await new Promise(r => setTimeout(r, 500));
  }
  throw new Error("Failed to get MXE public key");
}
```

## Computation Never Finalizes

1. Check callback instruction name: `#[arcium_callback(encrypted_ix = "flip")]`
2. Verify callback registered: `vec![FlipCallback::callback_ix(...)]`
3. Check Arx nodes running: `docker ps` shows arx containers
4. If the callback fires but `verify_output` returns `Err`, log the Anchor error and return `AbortedComputation`. Structured reasons such as `ExecutionFailure::CircuitFailure(...)` are recorded on the Arcium failure path, not returned by `verify_output`: `Err(e) => { msg!("Error: {}", e); return Err(ErrorCode::AbortedComputation.into()); }`

## Circuit Compilation Errors

| Error | Fix |
|-------|-----|
| Using `Vec`, `String`, `HashMap` | Use fixed-size arrays |
| Using `while`, `break`, `continue` | Use `for` with fixed bounds |
| Calling `.reveal()` inside conditional | Move reveal outside |
| Using `Option`, `Result` | Use struct with `is_some: bool` |

## Account Derivation Issues

Ensure PDAs match between program and client:

```typescript
// Client
const compDefAccount = getCompDefAccAddress(
  programId,
  Buffer.from(getCompDefAccOffset("flip")).readUInt32LE()
);

// Program uses same offset
const COMP_DEF_OFFSET_FLIP: u32 = comp_def_offset("flip");
```

## ArgBuilder Ordering Errors

### Symptom: Computation fails, as nodes aren't able to use data

ArgBuilder calls must match circuit fn parameter order left-to-right, with correct prefixes:

**For each `Enc<Shared, T>` parameter:**
1. `.x25519_pubkey(pub_key)`
2. `.plaintext_u128(nonce)`
3. `.encrypted_<type>(ct)` for each scalar in T

**For each `Enc<Mxe, T>` parameter:**
1. `.plaintext_u128(nonce)`
2. `.encrypted_<type>(ct)` for each scalar in T

**Common mistakes:**
- Forgetting `.x25519_pubkey()` for Shared params -- silent failure, most common bug
- Wrong parameter order -- must match circuit fn signature left-to-right
- Using wrong `encrypted_<type>` method (e.g., `encrypted_u64` for a u8 field)
- Passing combined `[u8; 64]` instead of two separate `[u8; 32]` calls (existing gotcha)

## Ciphertext Size Mismatch

**RescueCipher produces 32 bytes per value**, regardless of the original type:

| Encrypted Type | Scalar Count | Total Size | ArgBuilder Calls |
|----------------|--------------|------------|------------------|
| `bool`, `u8` | 1 | 32 bytes | 1x `.encrypted_u8([u8; 32])` |
| `u16`, `u32` | 1 | 32 bytes | 1x `.encrypted_u32([u8; 32])` |
| `u64` | 1 | 32 bytes | 1x `.encrypted_u64([u8; 32])` |
| `u128` | 1 | 32 bytes | 1x `.encrypted_u128([u8; 32])` |
| `(u64, u64)` | 2 | 64 bytes | 2x `.encrypted_u64([u8; 32])` |
| `{ a: u64, b: u64 }` | 2 | 64 bytes | 2x `.encrypted_u64([u8; 32])` |
| `[u32; 5]` | 5 | 160 bytes | 5x `.encrypted_u32([u8; 32])` |

**Formula**: `ciphertext_size = 32 * number_of_scalar_values`

**Critical**: Each scalar requires a **separate ArgBuilder call** with a `[u8; 32]` parameter. You cannot pass a combined `[u8; 64]` array for two values.

```rust
// WRONG - combined array, single call
pub fn add(ctx: Context<Add>, encrypted: [u8; 64], ...) -> Result<()> {
    let args = ArgBuilder::new()
        .encrypted_u64(encrypted)  // ERROR: expects [u8; 32]
        .build();
}

// CORRECT - separate arrays, multiple calls
pub fn add(ctx: Context<Add>, enc_a: [u8; 32], enc_b: [u8; 32], ...) -> Result<()> {
    let args = ArgBuilder::new()
        .encrypted_u64(enc_a)   // First value
        .encrypted_u64(enc_b)   // Second value
        .build();
}
```

**Common mistake**: Using `[u8; 8]` for u64 (wrong) instead of `[u8; 32]` (correct).

## Nonce Errors

### "Decryption failed" or Garbled Output

**Cause**: Nonce reuse or mismatch between encryption and decryption.

**Requirements**:
- Nonce must be **16 bytes**
- Nonce must be **unique** per (sharedSecret, plaintext) pair
- Same nonce must be passed to program as was used for encryption

```typescript
// WRONG: Reusing nonce for multiple encryptions
const nonce = randomBytes(16);
const ct1 = cipher.encrypt([value1], nonce);
const ct2 = cipher.encrypt([value2], nonce);  // BUG: same nonce!

// CORRECT: Fresh nonce for each encryption
const nonce1 = randomBytes(16);
const ct1 = cipher.encrypt([value1], nonce1);
const nonce2 = randomBytes(16);
const ct2 = cipher.encrypt([value2], nonce2);
```

**Note**: After MXE decrypts inputs, it increments the nonce by 1 before encrypting outputs. For subsequent computations, you must provide a fresh nonce.

## Localnet Issues

Localnet is auto-managed. There are no separate `start`/`stop` subcommands.

```bash
# Reset corrupted state
arcium clean    # Remove all localnet + build artifacts
arcium test     # Rebuilds and starts fresh localnet

# Run localnet without tests (stays running)
arcium localnet

# Keep localnet running after tests finish
arcium test --detach
```

### Speeding up localnet reruns (v0.10+)

**Targeted single test** (independent of keygen cache):

```bash
# Runs only tests/<name>.ts by temporarily rewriting Anchor.toml's [scripts] test glob.
# If the run is killed (Ctrl-C / SIGKILL), restore the original glob manually.
arcium test --test-name my_test
```

**Reusing cached MXE keys** with `--skip-keygen` — requires a populated cache first, or the CLI errors:

```
--skip-keygen requires a previous successful run: missing cache at <path>
Run `arcium test` (without `--skip-keygen`) at least once first.
```

The cache gets populated automatically at the end of every non-detached `arcium test` run (an internal `snapshot-mxe-keygen` call). Order:

```bash
# 1) Prime the cache (one-time, or after `arcium clean`). Non-detached.
arcium test

# 2) Now subsequent runs can skip keygen:
arcium test --skip-keygen
arcium localnet --skip-keygen
```

`--detach` does **not** auto-snapshot (the validator outlives the CLI). Snapshot yourself before teardown:

```bash
arcium test --detach
# ... interact with the running validator ...
arcium snapshot-mxe-keygen --rpc-url l   # `l` = localnet shorthand
# then tear down
```

`--skip-keygen` also exports `ARCIUM_SKIP_KEY_RECOVERY_INIT=1` into the test environment. Custom tests that unconditionally queue key-recovery-init must honor this flag or they'll fail on `--skip-keygen` reruns.

## CLI Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `arcium: command not found` | CLI not installed | `curl --proto '=https' --tlsv1.2 -sSfL https://install.arcium.com/ \| bash` or `arcup install` |
| `Error: Docker not running` | Docker daemon not started | Start Docker Desktop or `systemctl start docker` |
| Build fails with circuit errors | Arcis syntax issue | MCP: search "arcis syntax" for circuit reference |
| `anchor build` fails | Anchor/Solana version mismatch | Match the toolchain to [installation docs](https://docs.arcium.com/developers/installation) (current: Anchor 1.0.2, Solana 3.1.10); for CI, pin [`setup-arcium`](https://github.com/arcium-hq/setup-arcium) and set versions explicitly |

## Common Gotchas

### Account Offset Calculation Wrong

**Symptom**: Computation fails silently or returns garbage data
**Cause**: Wrong byte offset in `.account(pubkey, offset, size)`
**Fix**: Count ALL bytes before encrypted field:
- Discriminator: 8 bytes (Anchor adds this automatically)
- `bump: u8`: 1 byte
- `Pubkey`: 32 bytes
- `u64`: 8 bytes
- `u128`: 16 bytes

### Callback Account Not Written

**Symptom**: State not updated after callback completes
**Cause**: Writability not set in BOTH places
**Fix**: Set BOTH:
```rust
// 1. In callback_ix extra_accs when queuing
CallbackAccount { pubkey: ctx.accounts.game_state.key(), is_writable: true }

// 2. In callback accounts struct
#[account(mut)]  // <-- This is required!
pub game_state: Account<'info, GameState>,
```

### Circuit Hash Verification Failed

**Symptom**: "Hash verification failed" error on computation
**Cause**: Using placeholder `[0u8; 32]` instead of `circuit_hash!` for off-chain circuits
**Fix**: Always use the macro:
```rust
// WRONG
hash: [0u8; 32],

// CORRECT
use arcium_macros::circuit_hash;
hash: circuit_hash!("instruction_name"),
```

### Game State Invalid

**Symptom**: "Invalid game state" or unexpected state transitions
**Cause**: State machine transition without validation
**Fix**: Always validate state before transitioning:
```rust
require!(
    ctx.accounts.blackjack_game.game_state == GameState::PlayerTurn,
    ErrorCode::InvalidGameState
);
```

### Encrypted Output Wrong Size

**Symptom**: Deserialization fails or data corrupted
**Cause**: `LEN` parameter wrong in `SharedEncryptedStruct<LEN>` or `MXEEncryptedStruct<LEN>`
**Fix**: Count scalar values, not bytes:
- `u64` = 1 scalar
- `(u64, bool)` = 2 scalars
- `[u32; 5]` = 5 scalars
- `{ a: u64, b: u64 }` = 2 scalars

## Testing Patterns

### Unit Test Circuit Logic

Circuits are pure functions - test logic separately before encryption:

```rust
// In encrypted-ixs/src/lib.rs (outside #[encrypted] module)
#[cfg(test)]
mod tests {
    #[test]
    fn test_bid_comparison_logic() {
        // Test with known values - verifies algorithm before MPC
        let current_highest = 100u64;
        let new_bid = 150u64;
        assert!(new_bid > current_highest);
    }
}
```

### Integration Test with Localnet

```typescript
describe("my_app", () => {
  before(async () => {
    // `arcium test` auto-starts localnet with Arx nodes
  });

  it("initializes computation definition", async () => {
    await program.methods.initMyCompDef().rpc();
    // Verify comp def account exists
    const compDef = await program.account.compDef.fetch(compDefAddress);
    expect(compDef).to.exist;
  });

  it("queues and awaits computation", async () => {
    // Encrypt, queue, await finalization
    await program.methods.myInstruction(...).rpc();
    await awaitComputationFinalization(provider, offset, programId);
    // Verify callback executed via event or state change
  });
});
```

### Test State Transitions

```typescript
it("rejects invalid state transition", async () => {
  // Setup: set game state to RESOLVED
  // Action: attempt to call player_move (should fail)
  try {
    await program.methods.playerMove(...).rpc();
    assert.fail("Should have thrown InvalidGameState");
  } catch (e) {
    expect(e.message).to.include("InvalidGameState");
  }
});
```

### Debugging Failed Computations

| Check | Command/Action |
|-------|----------------|
| Arx nodes running | `docker ps | grep arx` |
| Computation queued | Check computation account exists |
| Callback registered | Verify `callback_ix` in queue call |
| Account writability | Both `CallbackAccount { pubkey, is_writable: true }` in `callback_ix` AND `#[account(mut)]` in callback struct |
| Comp def initialized | Call `init_*_comp_def` first |

### Test Encrypted State Updates

```typescript
it("updates encrypted state correctly", async () => {
  // Initial state
  await program.methods.initState().rpc();

  // Perform encrypted operation
  const nonce = randomBytes(16);
  const ciphertext = cipher.encrypt([BigInt(value)], nonce);
  await program.methods.updateState(Array.from(ciphertext[0]), ...).rpc();
  await awaitComputationFinalization(provider, offset, programId);

  // Verify state changed (may need to reveal or check side effects)
});
```

## See Also

- [Patterns](../examples/patterns.md) - Working code examples
- MCP: "arcis encrypted instruction" for circuit syntax
- MCP: "arcium_callback queue_computation" for program macros
- MCP: "RescueCipher encrypt" for client encryption
