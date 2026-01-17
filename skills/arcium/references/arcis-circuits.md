# Arcis Circuits

## Module Structure

```rust
use arcis::*;

#[encrypted]
mod circuits {
    use arcis::*;  // Required inside module too

    // Data structures
    pub struct MyInput { field: u8 }

    // Instructions (callable from Solana program)
    #[instruction]
    pub fn my_computation(input: Enc<Shared, MyInput>) -> OutputType {
        // ...
    }
}
```

**Key points:**
- `#[encrypted]` marks the module for MPC compilation
- `#[instruction]` marks callable entry points
- `use arcis::*;` needed both outside and inside module

## Encryption Types

`Enc<C, T>` = Encrypted value of type `T` with cipher context `C`

### Cipher Contexts

| Context | Who encrypts | Who decrypts | Use case |
|---------|--------------|--------------|----------|
| `Shared` | Client | Client | Client inputs, client-readable outputs |
| `Mxe` | MXE nodes | MXE nodes | Persistent on-chain state |

### Converting Between Encrypted and Secret-Shared

```rust
// Decrypt to secret shares (inside MPC only)
let plaintext = encrypted_input.to_arcis();

// Re-encrypt for same owner
let result = encrypted_input.owner.from_arcis(computed_value);

// Re-encrypt for MXE (persistent state)
let mxe_encrypted = Mxe.from_arcis(computed_value);
```

### Passing Multiple Owners

When function needs to output to different owners:
```rust
#[instruction]
pub fn multi_output(
    client: Shared,           // Output to client
    mxe: Mxe,                 // Output to MXE
    input: Enc<Shared, Data>,
) -> (Enc<Shared, Result>, Enc<Mxe, State>) {
    let data = input.to_arcis();
    (client.from_arcis(result), mxe.from_arcis(state))
}
```

## Supported Types

| Type | Notes |
|------|-------|
| `bool` | |
| `u8`, `u16`, `u32`, `u64`, `u128` | All unsigned integers |
| `i8`, `i16`, `i32`, `i64`, `i128` | All signed integers |
| `[T; N]` | Fixed-size arrays |
| Custom structs | All fields must be supported types |

**Not supported**: `Vec`, `String`, `HashMap`, `Option`, `Result`, enums, floats

## Randomness

```rust
// Random boolean
let coin_flip = ArcisRNG::bool();

// Shuffle array in place
let mut deck: [u8; 52] = [0, 1, 2, ...];
ArcisRNG::shuffle(&mut deck);
```

**Security**: Randomness is cryptographically secure, distributed across MPC nodes. No single node can predict or bias outcomes.

## Revealing Structs

To reveal an entire struct, call `.reveal()` on each field or construct a result struct:

```rust
pub struct AuctionResult {
    pub winner_lo: u128,
    pub winner_hi: u128,
    pub payment_amount: u64,
}

#[instruction]
pub fn determine_winner(state: Enc<Mxe, AuctionState>) -> AuctionResult {
    let s = state.to_arcis();
    AuctionResult {
        winner_lo: s.highest_bidder_lo,
        winner_hi: s.highest_bidder_hi,
        payment_amount: s.highest_bid,
    }.reveal()
}
```

## Common Patterns

### State Initialization

```rust
#[instruction]
pub fn init_state(mxe: Mxe) -> Enc<Mxe, MyState> {
    mxe.from_arcis(MyState { field: 0 })
}
```

### State Update

```rust
#[instruction]
pub fn update_state(
    input: Enc<Shared, Input>,
    state: Enc<Mxe, MyState>,
) -> Enc<Mxe, MyState> {
    let i = input.to_arcis();
    let mut s = state.to_arcis();
    s.field = i.value;
    state.owner.from_arcis(s)
}
```

### Conditional Logic

```rust
// Both branches execute - use assignment
let result = if secret_condition {
    expensive_calculation()
} else {
    cheap_calculation()
};
// Only `result` is used based on condition
```
