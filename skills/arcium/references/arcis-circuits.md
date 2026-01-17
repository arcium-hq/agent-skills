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

### Re-encrypting for Different Recipient (Sealing Pattern)

Re-encrypt data so a different party can decrypt it:

```rust
#[instruction]
pub fn re_encrypt_for_recipient(
    recipient: Shared,              // New owner who will decrypt
    data: Enc<Shared, SecretData>,  // Encrypted by original owner
) -> Enc<Shared, SecretData> {
    let d = data.to_arcis();        // Decrypt to secret shares
    recipient.from_arcis(d)         // Re-encrypt for recipient
}
```

**Use case**: A loan applicant submits encrypted financial data. After processing, the result is re-encrypted for the loan officer to view:

```rust
#[instruction]
pub fn check_eligibility(
    loan_officer: Shared,           // Will decrypt the result
    applicant_data: Enc<Shared, FinancialData>,
) -> Enc<Shared, EligibilityResult> {
    let data = applicant_data.to_arcis();
    let eligible = data.income > 50000 && data.credit_score > 700;
    loan_officer.from_arcis(EligibilityResult { approved: eligible })
}
```

## When to Use Each Encryption Context

| Scenario | Use | Why |
|----------|-----|-----|
| Client reads result | `Enc<Shared, T>` | Client can decrypt with shared secret |
| Persistent encrypted state | `Enc<Mxe, T>` | Only MXE can read/update across computations |
| Multi-computation state | `Enc<Mxe, T>` | Survives across independent computations |
| One-shot result | `Enc<Shared, T>` or `.reveal()` | Shared if private to client, reveal if public |

**Decision tree:**
1. Does client need to decrypt the result? -> `Enc<Shared, T>`
2. Does state persist across computations? -> `Enc<Mxe, T>`
3. Is result public to everyone? -> `.reveal()`

**Example - Voting:**
```rust
// Vote input: Enc<Shared, Vote> - client encrypts their vote
// Running tally: Enc<Mxe, Tally> - accumulates, never revealed until end
// Final result: .reveal() - public after voting ends

pub struct Vote { pub choice: bool }
pub struct Tally { pub yes: u64, pub no: u64 }

#[instruction]
pub fn cast_vote(vote: Enc<Shared, Vote>, tally: Enc<Mxe, Tally>) -> Enc<Mxe, Tally> {
    let v = vote.to_arcis();
    let mut t = tally.to_arcis();
    if v.choice { t.yes += 1 } else { t.no += 1 }
    tally.owner.from_arcis(t)  // Stays encrypted in MXE
}

#[instruction]
pub fn reveal_winner(tally: Enc<Mxe, Tally>) -> bool {
    let t = tally.to_arcis();
    (t.yes > t.no).reveal()  // Now public
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
| `f32`, `f64` | Fixed-point emulation (see below) |

**Not supported**: `Vec`, `String`, `HashMap`, `Option`, `Result`, enums

### Float Support

Floats are supported via fixed-point emulation:

```rust
let x: f64 = 3.14159;
let y: f64 = 2.71828;

// Arithmetic works as expected
let sum = x + y;
let product = x * y;

// Math methods available
let sqrt_x = x.sqrt();
let ln_x = x.ln();
let exp_x = x.exp();
```

**Available methods**: `.exp()`, `.exp2()`, `.ln()`, `.log2()`, `.sqrt()`

**Internal representation**: Values represented as k × 2^-52 where k ∈ [-2^250, 2^250]

**Performance**: Float operations are more expensive than integer operations due to fixed-point emulation overhead.

## Randomness

### ArcisRNG API

| Method | Return Type | Description |
|--------|-------------|-------------|
| `ArcisRNG::bool()` | secret `bool` | Random boolean |
| `ArcisRNG::gen_integer_from_width(width)` | secret integer | 0 to 2^width - 1 |
| `ArcisRNG::gen_public_integer_from_width(width)` | public integer | Public RNG (all nodes see same value) |
| `ArcisRNG::gen_integer_in_range(min, max, n_attempts)` | `(value, success)` | Value in [min, max], success bool |
| `ArcisRNG::shuffle(slice)` | `()` | Fisher-Yates shuffle in-place |

```rust
// Random boolean
let coin_flip = ArcisRNG::bool();

// Random u8 (0-255)
let random_byte = ArcisRNG::gen_integer_from_width(8);

// Random in range [1, 6] (dice roll) - may fail if unlucky
let (roll, success) = ArcisRNG::gen_integer_in_range(1, 6, 10);
// n_attempts = 10 means 10 rejection sampling attempts

// Shuffle array in place - O(n*log³(n) + n*log²(n)*sizeof(T))
let mut deck: [u8; 52] = [0, 1, 2, ...];
ArcisRNG::shuffle(&mut deck);
```

**`gen_integer_in_range` gotcha**: Returns `(value, success)` because rejection sampling may fail. Always check `success`:
```rust
let (value, ok) = ArcisRNG::gen_integer_in_range(0, 100, 20);
if !ok {
    // Fallback logic - all 20 attempts failed
}
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

## Control Flow Limitations

### Unsupported Constructs

| Construct | Why Not Supported |
|-----------|-------------------|
| `break`, `continue` | Would reveal iteration count |
| `return` (early) | Would reveal control flow path |
| `loop`, `while` | Unbounded iteration count |
| `match` | Not yet implemented |
| `if let` | Pattern matching not supported |

### Conditional Caveats

```rust
// WRONG: Missing else branch - compiler error
if condition {
    do_something()
}

// CORRECT: Both branches required
if condition {
    do_something()
} else {
    do_nothing()
}
```

**Important**: Both branches ALWAYS execute when condition is secret. No performance gain from "short-circuiting":
```rust
// Both expensive() and cheap() run - cost = both combined
let result = if secret_condition {
    expensive()  // Runs
} else {
    cheap()      // Also runs
};
// Only `result` is selected based on condition
```

### For Loops

Only fixed-bound `for` loops allowed:
```rust
// ALLOWED: Known iteration count
for i in 0..100 { ... }
for item in array.iter() { ... }

// NOT ALLOWED: Dynamic bounds
for i in 0..runtime_variable { ... }  // Compile error
```

See [Mental Model](mental-model.md) for why these constraints exist.

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

## See Also

- [Mental Model](mental-model.md) - Why MPC constraints exist (both branches execute, fixed loops, etc.)
- [Solana Integration](solana-program.md) - How circuits connect to Anchor programs
- [TypeScript Client](typescript-client.md) - Encrypting inputs client-side
- [Troubleshooting](troubleshooting.md) - Circuit compilation errors
