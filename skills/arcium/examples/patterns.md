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
        // ... similar for card_two, card_three
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

// Usage
let maybe = MaybeValue { value: 0, is_some: false };
if condition {
    maybe.value = computed;
    maybe.is_some = true;
}
```
