# Agari - Riichi Mahjong Scoring Engine

## Project Overview

**Agari** is a Rust-based Riichi Mahjong hand scoring engine. The goal is to build a complete scoring calculator that can:
- Parse a hand in standard notation (e.g., `123m456p789s11122z`)
- Decompose the hand into all valid meld structures
- Detect all applicable yaku (scoring patterns)
- Calculate han (doubles) and fu (minipoints)
- Output the final score/payout

This is a learning project—the developer knows Python but is new to Rust, so we're using a tutor/coach approach with incremental steps and comprehensive tests.

---

## What We Built (Steps 0-5)

### Step 1-2: Foundation
- **Tile representation**: `Tile`, `Suit`, `Honor` enums with traits for comparison and hashing
- **Parser**: Converts notation like `123m456p789s11z` into tile vectors
- **Red five support**: `0m`/`0p`/`0s` notation for akadora (parsed as 5s, tracked separately)
- **Validation**: 14-tile check, max 4 copies per tile

### Step 3: Hand Decomposition
- **`Meld` enum**: `Shuntsu(Tile)` for sequences, `Koutsu(Tile)` for triplets
- **`HandStructure` enum**: `Standard { melds, pair }` or `Chiitoitsu { pairs }`
- **`decompose_hand()`**: Generates ALL valid decompositions (important for scoring—same tiles can form different structures with different yaku)

### Step 4-5: Yaku Detection & Game Context
- **`GameContext` struct**: Complete game state for scoring
- **`detect_yaku_with_context()`**: Full yaku detection with context awareness
- **20 yaku implemented** including 2 yakuman (tenhou, chiihou)
- **Dora counting**: Regular dora, ura dora, and akadora
- **Open/closed handling**: Yaku validity and han reduction for open hands

---

## Current Architecture

```
agari/
├── Cargo.toml
└── src/
    ├── lib.rs           # Module declarations
    ├── tile.rs          # Tile, Suit, Honor enums
    ├── parse.rs         # Hand parsing, red fives, validation
    ├── hand.rs          # Meld, HandStructure, decompose_hand()
    ├── context.rs       # GameContext, WinType, dora calculation
    └── yaku.rs          # Yaku enum, detect_yaku_with_context()
```

**Tests passing:** 74

---

## Yaku Implemented (20 total)

### 1 Han
| Yaku | Notes |
|------|-------|
| Riichi | Closed only |
| Ippatsu | With riichi |
| MenzenTsumo | Closed + tsumo |
| Tanyao | All simples |
| Iipeikou | Closed only |
| Yakuhai(Honor) | Dragons always; winds if value wind |
| RinshanKaihou | Kan replacement win |
| HaiteiRaoyue | Last tile tsumo |
| HouteiRaoyui | Last tile ron |

### 2 Han
| Yaku | Notes |
|------|-------|
| DoubleRiichi | First turn riichi |
| Toitoi | All triplets |
| SanshokuDoujun | Same sequence in 3 suits (1 han open) |
| Ittsu | 123-456-789 one suit (1 han open) |
| Chiitoitsu | Seven pairs |
| Chanta | Terminal/honor in all groups (1 han open) |
| SanAnkou | Three concealed triplets |

### 3 Han
| Yaku | Notes |
|------|-------|
| Honitsu | One suit + honors (2 han open) |
| Junchan | Terminal in all groups (2 han open) |
| Ryanpeikou | Closed only |

### 6 Han
| Yaku | Notes |
|------|-------|
| Chinitsu | One suit only (5 han open) |

### Yakuman
| Yaku | Condition |
|------|-----------|
| Tenhou | Dealer wins on initial deal |
| Chiihou | Non-dealer wins on first draw |

---

## GameContext Structure

```rust
GameContext {
    win_type: WinType,          // Ron or Tsumo
    round_wind: Honor,          // East/South/West/North
    seat_wind: Honor,           // Player's wind
    is_open: bool,              // Has called tiles?
    is_riichi: bool,
    is_double_riichi: bool,
    is_ippatsu: bool,
    is_rinshan: bool,           // Won on kan replacement
    is_last_tile: bool,         // Haitei/Houtei
    is_tenhou: bool,            // Dealer first draw win
    is_chiihou: bool,           // Non-dealer first draw win
    dora_indicators: Vec<Tile>,
    ura_dora_indicators: Vec<Tile>,
    aka_count: u8,              // Red fives in hand
}

// Builder pattern usage:
let context = GameContext::new(WinType::Tsumo, Honor::East, Honor::South)
    .riichi()
    .ippatsu()
    .with_dora(vec![Tile::suited(Suit::Man, 1)])
    .with_aka(1);
```

---

## What's NOT Yet Implemented

### Yaku Missing
- **Pinfu** (requires wait type detection)
- **Chankan** (robbing a kan)
- **San Kantsu**, **Suu Kantsu** (3/4 kans—need kan representation)
- **Sanshoku Doukou** (same triplet in 3 suits)
- **Shousangen** (little three dragons)
- **Honroutou** (all terminals and honors)
- **More yakuman**: Kokushi, Suuankou, Daisangen, Shousuushii, Daisuushii, Tsuuiisou, Chinroutou, Ryuuiisou, Chuuren Poutou

### Core Features Missing
- **Fu calculation** (minipoints)
- **Score/payout calculation** (han + fu → points)
- **Wait type detection** (needed for pinfu, fu calculation)
- **Called melds representation** (chi/pon/kan as separate from closed melds)

---

## Next Steps (Planned)

### 1. Wait Type Detection & Pinfu
Detect the type of wait (what tile completed the hand):
- **Ryanmen** (two-sided): e.g., 23 waiting on 1 or 4
- **Kanchan** (middle): e.g., 13 waiting on 2
- **Penchan** (edge): e.g., 12 waiting on 3, or 89 waiting on 7
- **Shanpon** (dual pair): e.g., 11 22 waiting on either
- **Tanki** (single): e.g., pair wait

This enables:
- **Pinfu detection**: All sequences + valueless pair + ryanmen wait
- **Fu calculation**: Different waits have different fu values

### 2. Fu Calculation
Calculate minipoints based on:
- Base fu (20 for ron, 22 for tsumo, 25 for chiitoitsu)
- Meld fu (triplets/kans, open vs closed, terminals vs simples)
- Pair fu (value pair adds 2 fu)
- Wait fu (kanchan/penchan/tanki add 2 fu)
- Round up to nearest 10

### 3. Score Calculation
Convert han + fu to actual points:
- Lookup tables for common combinations
- Handle mangan, haneman, baiman, sanbaiman, yakuman thresholds
- Calculate dealer vs non-dealer payments
- Handle tsumo split vs ron single payment

### 4. CLI Interface
Create a user-friendly command-line interface:
```bash
$ agari "123m456p789s11122z" --tsumo --riichi --dora 1m

╔════════════════════════════════════════╗
║  Hand: 🀇🀈🀉 🀙🀚🀛 🀐🀑🀒 🀀🀀🀀 🀁🀁          ║
╠════════════════════════════════════════╣
║  Yaku:                                 ║
║    • Riichi (1 han)                    ║
║    • Menzen Tsumo (1 han)              ║
║    • Yakuhai: East (1 han)             ║
║                                        ║
║  Han: 3 + 1 dora = 4 han               ║
║  Fu: 40                                ║
║  Score: Mangan - 4000/2000             ║
╚════════════════════════════════════════╝
```

Features:
- Pretty-print tiles (Unicode mahjong characters or ASCII art)
- Display all context (dora, riichi status, win type, etc.)
- Show each yaku with its han value
- Show fu breakdown
- Show final score and payment structure

---

## Instructions for Resuming Development

Paste this into a new Claude chat:

---

**I'm building a Rust-based Riichi Mahjong hand scoring engine called "Agari". This is my first Rust project (I know Python). I'm working with a tutor/coach approach.**

**Project Goal:** A CLI scoring calculator that parses hands, detects yaku, calculates han/fu, and outputs pretty-printed results with score and payout.

**Completed so far (Steps 0-5):**
- Tile representation (`Suit`, `Honor`, `Tile` enums)
- Parser with red five support (`0m`/`0p`/`0s` notation)
- Hand decomposition (generates all valid `HandStructure`s)
- 20 yaku detection with full `GameContext` support
- Situational yaku: riichi, ippatsu, double riichi, rinshan, haitei, houtei, tenhou, chiihou
- Dora counting (regular + ura + akadora)
- Open/closed hand handling with han reduction

**Tests passing:** 74

**Next steps I want to tackle:**
1. **Wait type detection** — Determine what tile completed the hand (ryanmen, kanchan, penchan, tanki, shanpon) to enable Pinfu detection and fu calculation
2. **Fu calculation** — Minipoints based on melds, pair, wait type
3. **Score calculation** — Han + fu → points, with mangan/haneman thresholds, dealer vs non-dealer, tsumo vs ron payouts
4. **CLI interface** — Pretty terminal output showing hand, yaku list, han, fu, score, and payout

**Please continue coaching me through the next step (wait type detection and Pinfu).**

---

**Attached files:** (attach the src/ folder contents)
- `lib.rs`
- `tile.rs`
- `parse.rs`
- `hand.rs`
- `context.rs`
- `yaku.rs`
- `Cargo.toml`

---

## File Reference

The source files are available in the `agari_src/` folder. Key entry points:

- **Parsing a hand:** `parse::parse_hand_with_aka("123m456p789s11122z")`
- **Converting to counts:** `parse::to_counts(&tiles)`
- **Decomposing:** `hand::decompose_hand(&counts)`
- **Detecting yaku:** `yaku::detect_yaku_with_context(&structure, &counts, &context)`

Good luck with the next phase! 🀄