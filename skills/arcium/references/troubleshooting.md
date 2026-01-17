# Troubleshooting

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| "Failed to fetch MXE public key" | ARX nodes not ready | Add retry with exponential backoff |
| Computation never finalizes | Missing/incorrect callback | Verify `callback_ix` registration |
| Type mismatch in circuit | `Enc` owner inconsistency | Match `Shared`/`Mxe` throughout |
| "Stack height exceeded" | Too many CPIs | Reduce instruction nesting |
| "Account not initialized" | Missing `init_comp_def` | Call initialization first |
| "Invalid cluster" | Wrong network | Verify cluster offset |

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
3. Check ARX nodes running: `docker ps` shows arx containers

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

## Ciphertext Size Mismatch

Match instruction parameter size to encrypted data:

| Encrypted Type | Ciphertext Size |
|----------------|-----------------|
| `u8`, `bool` | 32 bytes |
| `u64` | 32 bytes |
| Struct with 2x u64 | 64 bytes |

## Localnet Issues

```bash
# Reset if state is corrupted
arcium localnet stop
arcium localnet start

# Or use fresh test run
arcium test  # Auto-manages localnet
```
