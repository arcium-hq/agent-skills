# TypeScript Client

## Dependencies

```json
{
  "@arcium-hq/client": "^0.6.2",
  "@coral-xyz/anchor": "^0.32.1"
}
```

## Imports

```typescript
import {
  getMXEPublicKey,
  awaitComputationFinalization,
  getArciumEnv,
  getCompDefAccOffset,
  RescueCipher,
  deserializeLE,
  getMXEAccAddress,
  getMempoolAccAddress,
  getCompDefAccAddress,
  getExecutingPoolAccAddress,
  x25519,
  getComputationAccAddress,
  getClusterAccAddress,
} from "@arcium-hq/client";
```

## Key Exchange with Retry

ARX nodes may not be ready immediately. Always use retry - see [MXE Public Key is Null](troubleshooting.md#mxe-public-key-is-null) for the retry pattern.

## Key Generation

### Production: Deterministic Keys from Wallet Signature

For production apps, derive keys deterministically from wallet signatures so users can recover their encryption keys:

```typescript
import { sha256 } from "@noble/hashes/sha256";

// PRODUCTION: Derive deterministic keys from wallet signature
async function deriveEncryptionKeys(
  wallet: { signMessage: (msg: Uint8Array) => Promise<Uint8Array> },
  programId: PublicKey
): Promise<{ privateKey: Uint8Array; publicKey: Uint8Array }> {
  const message = `Arcium encryption key for ${programId.toBase58()}`;
  const signature = await wallet.signMessage(new TextEncoder().encode(message));
  const privateKey = sha256(signature).slice(0, 32); // X25519 seed
  const publicKey = x25519.getPublicKey(privateKey);
  return { privateKey, publicKey };
}

const { privateKey, publicKey } = await deriveEncryptionKeys(wallet, programId);
```

### Testing: Ephemeral Random Keys

```typescript
// TESTING ONLY: Random keys (don't use in production - keys are lost on refresh)
const privateKey = x25519.utils.randomSecretKey();
const publicKey = x25519.getPublicKey(privateKey);
```

## Client-Side Encryption with RescueCipher

```typescript
const sharedSecret = x25519.getSharedSecret(privateKey, mxePublicKey);
const cipher = new RescueCipher(sharedSecret);

const nonce = randomBytes(16);  // From 'crypto' (Node.js) or @noble/hashes
const ciphertext = cipher.encrypt([BigInt(value)], nonce);
```

### RescueCipher: 32 Bytes Per Value

**Every encrypted value produces exactly 32 bytes**, regardless of the original type:

| Original Type | Encrypted Size |
|---------------|----------------|
| `bool`, `u8` | 32 bytes |
| `u64` | 32 bytes |
| `u128` | 32 bytes |
| Two values | 64 bytes (32 * 2) |
| N values | 32 * N bytes |

### Nonce Requirements

- **Size**: 16 bytes
- **Uniqueness**: Must be unique per (sharedSecret, plaintext) pair
- Use `randomBytes(16)` for each new encryption

```typescript
// Each encryption needs a fresh nonce
const nonce1 = randomBytes(16);
const ct1 = cipher.encrypt([value1], nonce1);

const nonce2 = randomBytes(16);  // Different nonce!
const ct2 = cipher.encrypt([value2], nonce2);
```

### Nonce Increment for Outputs (Critical)

**After MXE decrypts inputs with nonce N, it uses nonce N+1 for encrypting outputs.**

```
Client encrypts:  nonce = N
MXE decrypts:     uses nonce N
MXE encrypts output: uses nonce N+1
```

**Why this matters for decryption:**
```typescript
// If you receive encrypted output, decrypt with nonce+1
const outputNonce = Buffer.alloc(16);
outputNonce.writeBigUInt64LE(deserializeLE(originalNonce) + 1n, 0);
const decryptedOutput = cipher.decrypt(encryptedOutput, outputNonce);
```

**Note**: This is handled automatically when using `awaitComputationFinalization()` with revealed outputs. Only relevant when decrypting `Enc<Shared, T>` outputs manually.

## Transaction Submission

```typescript
const computationOffset = new anchor.BN(randomBytes(8), "hex");

await program.methods
  .flip(
    computationOffset,
    Array.from(ciphertext[0]),
    Array.from(publicKey),
    new anchor.BN(deserializeLE(nonce).toString())
  )
  .accountsPartial({
    computationAccount: getComputationAccAddress(
      getArciumEnv().arciumClusterOffset,
      computationOffset
    ),
    clusterAccount: getClusterAccAddress(getArciumEnv().arciumClusterOffset),
    mxeAccount: getMXEAccAddress(programId),
    mempoolAccount: getMempoolAccAddress(getArciumEnv().arciumClusterOffset),
    executingPool: getExecutingPoolAccAddress(getArciumEnv().arciumClusterOffset),
    compDefAccount: getCompDefAccAddress(
      programId,
      Buffer.from(getCompDefAccOffset("flip")).readUInt32LE()
    ),
  })
  .rpc({ skipPreflight: true, commitment: "confirmed" });
```

## Await Result

```typescript
const finalizeSig = await awaitComputationFinalization(
  provider,
  computationOffset,
  programId,
  "confirmed"
);
```

## Event Listening

```typescript
type Event = anchor.IdlEvents<typeof program["idl"]>;

const awaitEvent = async <E extends keyof Event>(eventName: E): Promise<Event[E]> => {
  let listenerId: number;
  const event = await new Promise<Event[E]>((res) => {
    listenerId = program.addEventListener(eventName, (event) => res(event));
  });
  await program.removeEventListener(listenerId);
  return event;
};

const flipEvent = await awaitEvent("flipEvent");  // Must match your #[event] struct name
console.log("Result:", flipEvent.result);
```

## Multiple Values Encryption

To encrypt multiple values (e.g., a struct with two u64 fields):

```typescript
const a = BigInt(10);
const b = BigInt(32);
const ciphertext = cipher.encrypt([a, b], nonce);

// Flatten for Solana instruction
const flatCiphertext = Array.from(ciphertext.flat());
```

## See Also

- [Solana Integration](solana-program.md) - `ArgBuilder` methods and ciphertext size mapping
- [Arcis Circuits](arcis-circuits.md) - Input struct types matching client encryption
- [Troubleshooting](troubleshooting.md) - Nonce errors and ciphertext size mismatch
