# MPC Mental Model

## How Secret Sharing Works

When `.to_arcis()` is called, data isn't decrypted. It's converted to **secret shares** distributed across ARX nodes:

```
Secret value: 42

Node A holds: 17  (random)
Node B holds: 93  (random)
Node C holds: -68 (calculated so shares sum to 42)
```

**Key insight**: Each node sees only random-looking data. No single node learns the original value. When nodes compute together following the MPC protocol, the math works out correctly.

**Security**: All nodes would need to collude to reconstruct secrets. One honest node protects everything.

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

**Why**: Both branches execute in isolation before merging. `.reveal()` is a global side effect that cannot be undone.

## Cost Model

| Operation | Cost | Notes |
|-----------|------|-------|
| Addition, subtraction | Nearly free | Local computation on shares |
| Multiply by constant | Nearly free | Local scalar multiply |
| Multiply two secrets | Cheap | Optimized via preprocessing |
| Comparisons (`<`, `>`, `==`) | Expensive | Bit-by-bit operations |
| Division, modulo | Very expensive | Iterative algorithms |
| Dynamic array indexing | O(n) | Must check all positions |

**Optimization**: Batch `.from_arcis()` calls - restructure to minimize conversions.
