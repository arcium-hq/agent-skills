# Common Patterns

## 1. Stateless Computation (Coinflip)

No persistent state. Input -> Compute -> Reveal.

```rust
#[instruction]
pub fn flip(input: Enc<Shared, UserChoice>) -> bool {
    let input = input.to_arcis();
    let toss = ArcisRNG::bool();
    (input.choice == toss).reveal()
}
```

**Source**: `examples/coinflip/encrypted-ixs/src/lib.rs`

---

## 2. Persistent Encrypted State (Voting)

State encrypted with `Mxe` persists across computations.

```rust
pub struct VoteStats { yes: u64, no: u64 }

#[instruction]
pub fn init_vote_stats(mxe: Mxe) -> Enc<Mxe, VoteStats> {
    mxe.from_arcis(VoteStats { yes: 0, no: 0 })
}

#[instruction]
pub fn vote(
    vote: Enc<Shared, UserVote>,
    stats: Enc<Mxe, VoteStats>,
) -> Enc<Mxe, VoteStats> {
    let v = vote.to_arcis();
    let mut s = stats.to_arcis();
    if v.vote { s.yes += 1 } else { s.no += 1 }
    stats.owner.from_arcis(s)
}

#[instruction]
pub fn reveal_result(stats: Enc<Mxe, VoteStats>) -> bool {
    let s = stats.to_arcis();
    (s.yes > s.no).reveal()
}
```

**Source**: `examples/voting/encrypted-ixs/src/lib.rs`

---

## 3. Multi-Party Input (Rock Paper Scissors)

Multiple parties submit encrypted inputs.

```rust
pub struct GameMoves { player_a_move: u8, player_b_move: u8 }
pub struct PlayersMove { player: u8, player_move: u8 }

#[instruction]
pub fn player_move(
    move_ctxt: Enc<Shared, PlayersMove>,
    game_ctxt: Enc<Mxe, GameMoves>,
) -> Enc<Mxe, GameMoves> {
    let m = move_ctxt.to_arcis();
    let mut g = game_ctxt.to_arcis();

    if m.player == 0 && g.player_a_move == 3 && m.player_move < 3 {
        g.player_a_move = m.player_move;
    } else if m.player == 1 && g.player_b_move == 3 && m.player_move < 3 {
        g.player_b_move = m.player_move;
    }

    game_ctxt.owner.from_arcis(g)
}
```

**Source**: `examples/rock_paper_scissors/against-player/encrypted-ixs/src/lib.rs`

---

## 4. Randomness (Blackjack)

Cryptographically secure randomness via `ArcisRNG`.

```rust
#[instruction]
pub fn shuffle_and_deal_cards(mxe: Mxe, client: Shared) -> (...) {
    let mut deck = INITIAL_DECK;
    ArcisRNG::shuffle(&mut deck);

    // Deal cards...
    mxe.from_arcis(Deck::from_array(deck))
}
```

**Source**: `examples/blackjack/encrypted-ixs/src/lib.rs`

---

## 5. Complex Comparison (Sealed-Bid Auction)

Track highest/second-highest with encrypted comparisons.

```rust
pub struct AuctionState {
    highest_bid: u64,
    highest_bidder_lo: u128,
    highest_bidder_hi: u128,
    second_highest_bid: u64,
}

#[instruction]
pub fn place_bid(
    bid: Enc<Shared, Bid>,
    state: Enc<Mxe, AuctionState>,
) -> Enc<Mxe, AuctionState> {
    let b = bid.to_arcis();
    let mut s = state.to_arcis();

    if b.amount > s.highest_bid {
        s.second_highest_bid = s.highest_bid;
        s.highest_bid = b.amount;
        s.highest_bidder_lo = b.bidder_lo;
        s.highest_bidder_hi = b.bidder_hi;
    } else if b.amount > s.second_highest_bid {
        s.second_highest_bid = b.amount;
    }

    state.owner.from_arcis(s)
}
```

**Source**: `examples/sealed_bid_auction/encrypted-ixs/src/lib.rs`

---

## 6. Efficient Packing (Blackjack Deck)

Pack 52 cards into 3 u128s for efficient on-chain storage.

```rust
pub struct Deck {
    pub card_one: u128,   // Cards 0-20 (21 cards x 6 bits)
    pub card_two: u128,   // Cards 21-41
    pub card_three: u128, // Cards 42-51
}

impl Deck {
    pub fn from_array(array: [u8; 52]) -> Deck {
        let mut card_one = 0u128;
        for i in 0..21 {
            card_one += POWS_OF_SIXTY_FOUR[i] * array[i] as u128;
        }
        // card_two and card_three follow same pattern (indices 21-41 and 42-51)
        // Full implementation: github.com/arcium-hq/examples/tree/main/blackjack
        Deck { card_one, card_two, card_three }
    }
}
```

**Source**: `examples/blackjack/encrypted-ixs/src/lib.rs`

---

## 7. Boolean Flag Pattern

Simulate early exit without `break`:

```rust
let mut found = false;
for i in 0..MAX_ITEMS {
    if !found && items[i] == target {
        found = true;
        result_index = i;
    }
}
```

---

## 8. Option-Like Pattern

Replace `Option<T>` with explicit flag:

```rust
pub struct MaybeValue {
    pub value: u64,
    pub is_some: bool,
}

// Setting a value
let mut maybe = MaybeValue { value: 0, is_some: false };
if condition {
    maybe.value = computed;
    maybe.is_some = true;
}

// Reading the value (check flag first)
if maybe.is_some {
    let val = maybe.value;  // Safe to use
}
```

---

## 9. State Machine Pattern (Blackjack)

Use enums for game state, validate transitions before processing:

```rust
// In Solana program (not circuit - enums not supported in Arcis)
pub enum GameState {
    Initial = 0,
    PlayerTurn = 1,
    DealerTurn = 2,
    Resolving = 3,
    Resolved = 4,
}

#[account]
pub struct Game {
    pub game_state: u8,  // Store as u8
    pub encrypted_deck: [u8; 96],
    // ...
}

// Always validate state before transition
pub fn player_hit(ctx: Context<PlayerHit>) -> Result<()> {
    require!(
        ctx.accounts.game.game_state == GameState::PlayerTurn as u8,
        ErrorCode::InvalidGameState
    );
    // ... queue computation
}
```

**Source**: `examples/blackjack/programs/blackjack/src/lib.rs`

---

## 10. Pubkey as u128 Pair (Sealed Auction)

Solana pubkeys are 32 bytes, but Arcis only supports up to u128. Split into two u128s:

```rust
// Circuit struct
pub struct Bid {
    pub amount: u64,
    pub bidder_lo: u128,  // First 16 bytes of pubkey
    pub bidder_hi: u128,  // Last 16 bytes of pubkey
}
```

```typescript
// Client-side splitting
function splitPubkeyToU128s(pubkey: Uint8Array): { lo: bigint; hi: bigint } {
    const loBytes = pubkey.slice(0, 16);
    const hiBytes = pubkey.slice(16, 32);
    return {
        lo: deserializeLE(loBytes),
        hi: deserializeLE(hiBytes)
    };
}

// Reconstruct
function u128sToPubkey(lo: bigint, hi: bigint): PublicKey {
    const loBytes = serializeLE(lo, 16);
    const hiBytes = serializeLE(hi, 16);
    return new PublicKey(Buffer.concat([loBytes, hiBytes]));
}
```

**Source**: `examples/sealed_bid_auction/tests/sealed_bid_auction.ts`

---

## 11. Multiple Nonces Pattern (Blackjack)

When tracking different encrypted states for different parties:

```rust
#[account]
pub struct Game {
    pub deck_nonce: u128,    // For deck encryption
    pub client_nonce: u128,  // For player hand
    pub dealer_nonce: u128,  // For dealer hand
}
```

Each encryption context needs its own nonce to prevent correlation attacks.

**Source**: `examples/blackjack/programs/blackjack/src/lib.rs`

---

## 12. Combining Multiple Patterns

Complex apps combine patterns. Here's a framework for architecting multi-pattern apps.

### Example: Sealed-Bid Auction Architecture

| Pattern | Purpose | Implementation |
|---------|---------|----------------|
| Persistent State (#2) | Store bids | `Enc<Mxe, AuctionState>` |
| Multi-Party Input (#3) | Each bidder submits | Multiple `Enc<Shared, Bid>` inputs |
| Comparisons (#5) | Find highest bid | Compare encrypted bids |
| State Machine (#9) | Auction phases | `status: u8` with validation |

### Architecture Steps

1. **Define state struct** with all encrypted fields:
   ```rust
   pub struct AuctionState {
       highest_bid: u64,
       highest_bidder: u128,
       status: u8,  // 0=open, 1=closed, 2=revealed
   }
   ```

2. **Map out computation phases** (init -> bidding -> resolution):
   - `init_auction`: Create `Enc<Mxe, AuctionState>` with zero values
   - `place_bid`: Accept `Enc<Shared, Bid>`, compare, update state
   - `close_auction`: Validate status, set to closed
   - `reveal_winner`: `.reveal()` winner after close

3. **Choose encryption context per field** ([see decision tree](../references/arcis-circuits.md#when-to-use-each-encryption-context)):
   - Bid amounts: `Enc<Mxe, T>` (persists across bids, never revealed to clients)
   - Final winner: `.reveal()` (public after auction ends)

4. **Add state validation** before each transition (Solana program side):
   ```rust
   require!(auction.status == 0, ErrorCode::AuctionClosed);
   ```

5. **Plan account creation** (init) vs updates (callbacks):
   - Init: Creates account + queues first computation
   - Updates: Callback writes back to existing account

### Common Pattern Combinations

| App Type | Patterns Used |
|----------|---------------|
| Game (poker, blackjack) | Persistent State + Randomness + Multi-Party + State Machine |
| DeFi (auctions, AMM) | Persistent State + Comparisons + Multi-Party |
| Voting | Multi-Party + Aggregation + Reveal at end |
| Identity (credit check) | Re-encryption (Sealing) + Comparisons |

---

**Note**: All source paths (e.g., `examples/blackjack/...`) reference the [Arcium Examples Repository](https://github.com/arcium-hq/examples).

## See Also

- [Mental Model](../references/mental-model.md) - Why these patterns are needed
- [Arcis Circuits](../references/arcis-circuits.md) - Type constraints
- [Troubleshooting](../references/troubleshooting.md) - Common pattern mistakes
