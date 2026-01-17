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

ARX nodes may not be ready immediately. Always use retry:

```typescript
async function getMXEPublicKeyWithRetry(
  provider: AnchorProvider,
  programId: PublicKey,
  maxRetries = 20,
  retryDelayMs = 500
): Promise<Uint8Array> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const key = await getMXEPublicKey(provider, programId);
      if (key) return key;
    } catch (error) {
      console.log(`Attempt ${attempt} failed`);
    }
    await new Promise(r => setTimeout(r, retryDelayMs));
  }
  throw new Error(`Failed after ${maxRetries} attempts`);
}

const mxePublicKey = await getMXEPublicKeyWithRetry(provider, programId);
```

## Encryption

```typescript
const privateKey = x25519.utils.randomSecretKey();
const publicKey = x25519.getPublicKey(privateKey);
const sharedSecret = x25519.getSharedSecret(privateKey, mxePublicKey);
const cipher = new RescueCipher(sharedSecret);

const nonce = randomBytes(16);
const ciphertext = cipher.encrypt([BigInt(value)], nonce);
```

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

const flipEvent = await awaitEvent("flipEvent");
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
