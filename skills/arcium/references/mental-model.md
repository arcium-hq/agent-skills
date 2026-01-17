# MPC Mental Model

## ARX Nodes and Secret Sharing

**ARX nodes** (Arcium eXecution nodes) are the distributed network that executes encrypted computations. They run MPC protocols to compute on data without any single node seeing the actual values.

**MXE** (MPC Execution Environment) is the runtime that coordinates these nodes and manages the encrypted computation lifecycle.

When `.to_arcis()` is called, data isn't decrypted. It's converted to **secret shares** distributed across ARX nodes:

```
Secret value: 42

Node A holds: 17  (random)
Node B holds: 93  (random)
Node C holds: 188 (calculated: 42 - 17 - 93 + 256 = 188 mod 256)
              ↑ Shares computed in finite field arithmetic
```

**Key insight**: Each node sees only random-looking data. No single node learns the original value. When nodes compute together following the MPC protocol, the math works out correctly.

**Security Threshold (n-of-n)**: Arcium uses n-of-n secret sharing - **all** nodes would need to collude to reconstruct secrets. This means a single honest node protects everything. Even if n-1 nodes are compromised, your data remains private.

```rust
#[instruction]
pub fn double_secret(input: Enc<Shared, u64>) -> Enc<Shared, u64> {
    let value = input.to_arcis();      // Secret shares across nodes
    let result = value * 2;            // Each node multiplies their share
    input.owner.from_arcis(result)     // Re-encrypt
}
```

## The Circuit is Fixed at Compile Time

Arcis code compiles into a **fixed circuit structure** before data flows through:

```
Rust Code -> [compile] -> Fixed Circuit -> [runtime] -> Secret Shares Flow Through
```

The circuit structure (operations, order, iteration count) is locked at compile time.

**This is the root cause of all constraints.** If structure could change based on secret data, observers could learn information by watching HOW computation runs.

## Both Branches Always Execute

**Normal Rust:**
```rust
if condition { expensive() } else { cheap() }
// ONE branch executes
```

**Arcis (when condition is secret):**
```rust
let secret = input.to_arcis();
let is_large = secret > 1000;  // is_large is also secret-shared

if is_large {
    expensive()  // Always runs
} else {
    cheap()      // Always runs
}
// Cost = cost(expensive) + cost(cheap)
```

**Why**: No single node knows `is_large` (it's secret-shared). Protocol must compute both branches, then select the correct result.

**Exception**: Compile-time constants like `if true { ... }` or `if CONST > 5 { ... }` allow single-branch execution.

## Fixed Iteration Counts

**Allowed:**
```rust
for i in 0..100 { process(data[i]); }  // Count known at compile time
```

**Not allowed:**
```rust
while secret_value < threshold { ... }  // Count depends on secret
for i in 0..runtime_variable { ... }    // Unknown at compile time
```

**Why no break/continue?** Early exit reveals information:
```rust
for i in 0..1000 {
    if found_match { break; }  // Would reveal when match occurred
}
```

**Pattern for "early exit" behavior:**
```rust
let mut done = false;
for i in 0..MAX {
    if !done {
        // ... work ...
        if condition { done = true; }
    }
}
```

## Fixed-Size Data Only

**Not supported:**
```rust
let items: Vec<u8> = vec![];
let name: String = String::new();
```

**Use instead:**
```rust
let items: [u8; 100] = [0; 100];
let name: [u8; 32] = [0; 32];
```

**Why**: Circuit compiler must know exactly how much memory and operations needed. Variable-size creates unknown circuit size.

**Patterns:**
- Fixed arrays with length counter: `([T; MAX], usize)`
- Sentinel values: `u8::MAX` for "empty"
- Option-like: `{ value: T, is_some: bool }`

## Reveal and Encryption Placement

`.reveal()` and `.from_arcis()` **cannot be called inside if/else** when condition is secret:

**Wrong:**
```rust
if secret_condition { value.reveal() }  // Compiler error
```

**Correct:**
```rust
let selected = if secret_condition { a } else { b };
selected.reveal()  // Outside conditional
```

**Why `.reveal()` can't be conditional**:
1. Both branches execute (no node knows which is "real")
2. `.reveal()` is a **global side effect** - it triggers network communication between nodes
3. If reveal were in a branch, it would either: (a) reveal in both branches (leaking both values), or (b) reveal in neither (breaking the protocol)
4. The MPC protocol has no way to "undo" a reveal or make it conditional

## Cost Model

### Performance Hierarchy

```
Cheapest ──────────────────────────────────► Most Expensive

  Additions       Multiplications       Comparisons
  (basically      (expensive due to     (convert to bits,
   free)          preprocessing)         element-wise compare)
```

| Operation | Relative Cost | Notes |
|-----------|---------------|-------|
| Addition, subtraction | 1x | Local computation on shares |
| Multiply by constant | 1x | Local scalar multiply |
| Multiply two secrets | 10x | Requires heavy preprocessing |
| Comparisons (`<`, `>`, `==`) | ~50x | Bit-by-bit comparison protocols |
| Division, modulo | ~100x | Iterative algorithms |
| Dynamic array indexing | O(n) | Must check all positions |

**Why multiplications are expensive**: MPC multiplication requires "preprocessing" - nodes must coordinate to generate correlated randomness before the actual computation. This is done in batches but still dominates circuit cost.

**Why comparisons are even more expensive**: Comparing two secret values requires:
1. Convert both to bit representation
2. Compare bit-by-bit (each bit comparison = multiple multiplications)
3. Combine results

**Optimization tips**:
- Batch `.from_arcis()` calls - restructure to minimize conversions
- Prefer arithmetic over comparisons (e.g., sum instead of count-if)
- Pre-compute comparison results if used multiple times
- Use `u64` instead of `u128` when possible (fewer bits = cheaper comparisons)

## Security Considerations

### Trust Model

- **n-of-n threshold**: ALL ARX nodes must collude to reconstruct secrets
- Single honest node protects everything - even if n-1 nodes are compromised
- Nodes see circuit structure (WHAT computation runs) but not data values

### What's Safe to Reveal

| Safe | Why |
|------|-----|
| Public parameters (timestamps, counts) | No private info |
| Final results after computation completes | Computation done |
| Aggregate statistics (totals, averages) | Individual inputs hidden |

### What to Keep Encrypted

| Keep Encrypted | Why |
|----------------|-----|
| Individual inputs (votes, bids, cards) | Would reveal user choices |
| Intermediate state during multi-step protocols | Could leak partial info |
| Anything identifying users | Privacy |

### Gotchas

**Circuit structure is visible**: Adversary knows WHAT computation runs, but not on what data. If circuit structure reveals intent (e.g., "this user called bid_higher"), consider batching or dummy operations.

**Timing can leak info**: If computation time varies based on secret data, timing attacks possible. Use constant-time patterns (both branches always execute).

**Reveal placement matters**: `.reveal()` outside conditionals always runs (see [Reveal and Encryption Placement](#reveal-and-encryption-placement)).

## See Also

- [Arcis Circuits](arcis-circuits.md) - Circuit syntax and `Enc<C,T>` types
- [Solana Integration](solana-program.md) - Anchor program integration
- [Troubleshooting](troubleshooting.md) - Common errors from MPC constraints
