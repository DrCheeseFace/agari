---
title: Untitled
author: Ryan Sullenberger
date: 2026-01-10T21:08:43Z
---

# RIICHI SCORING TOOL — MVP WORK PLAN

## PROJECT DEFINITION (read once, then start coding)

**MVP Definition**

> Input: 14 tiles (any order) + minimal context
> Output: best-scoring interpretation, list of yaku, total han
>
> No fu, no payments, closed hands only

---

## STEP 0 — Ruleset & guardrails (½ day)

### Goal

Freeze assumptions so you don’t second-guess yourself later.

### Tasks

* Use standard Japanese Riichi rules
* Closed hands only
* No yakuman
* No fu
* No open melds
* No red fives
* No kans

### Deliverable

* `RULES.md` with bullet points above

### Do NOT:

* Add “just one more yaku”
* Support house rules

---

## STEP 1 — Tile representation & validation

### Goal

Accept unordered tile input and validate legality.

### Tasks

1. Define canonical tile type
2. Parse input into tile counts
3. Validate:

   * exactly 14 tiles
   * legal tiles
   * no tile count > 4

### Deliverable

```ts
parseTiles(input: string[]): TileCounts
```

### Acceptance test

* Agari hand input, ideally would be something like: 123m456p789p789s11z, and the parser can handle this
* Ultimately, we need to be able to indicate which part of the hand is open vs closed. We should think about the best way to do this.

### Do NOT:

* Think about melds
* Think about scoring

---

## STEP 2 — Hand completeness checker

### Goal

Determine if a hand is **complete**, regardless of scoring.

### Tasks

1. Implement chiitoitsu check
2. Implement standard hand check:

   * try every possible pair
   * recursively remove melds

### Deliverable

```ts
isWinningHand(counts: TileCounts): boolean
```

### Acceptance test

* Correctly accepts/rejects known winning hands

### Do NOT:

* Detect yaku
* Pick “best” structure

---

## STEP 3 — Hand decomposition engine (core)

### Goal

Generate **all valid hand structures**.

### Tasks

1. For each valid pair:

   * clone counts
   * remove pair
   * recursively extract melds
2. Record each successful decomposition

### Deliverable

```ts
getHandStructures(counts: TileCounts): HandStructure[]
```

Where:

```ts
HandStructure {
  melds: Meld[]
  pair: Tile
}
```

### Acceptance test

* Ambiguous hands return multiple structures
* No false positives

### Do NOT:

* Collapse to one structure
* Score anything yet

---

## STEP 4 — Winning tile & wait type detection

### Goal

Determine how the hand was completed.

### Tasks

1. Require explicit winning tile input
2. For each HandStructure:

   * identify which meld was completed
   * classify wait type

### Deliverable

```ts
annotateWaitType(
  structure: HandStructure,
  winningTile: Tile
): HandStructureWithWait
```

### Acceptance test

* Correctly identifies ryanmen / kanchan / penchan / tanki / shanpon

### Do NOT:

* Calculate fu
* Special-case pinfu yet

---

## STEP 5 — Minimal context model

### Goal

Add game state without entangling logic.

### Tasks

Define:

```ts
Context {
  winType: "ron" | "tsumo"
  seatWind: Wind
  roundWind: Wind
  riichi: boolean
}
```

### Deliverable

* Context object passed into yaku evaluation

### Do NOT:

* Add ippatsu, rinshan, etc.

---

## STEP 6 — MVP yaku detection

### Goal

Detect common yaku using pure functions.

### Tasks

Implement these yaku **only**:

* Riichi
* Menzen Tsumo
* Pinfu
* Tanyao
* Yakuhai (seat / round / dragons)
* Iipeikou
* Toitoi
* Chiitoitsu

Each yaku:

```ts
detectYaku(hand: HandStructureWithWait, ctx: Context): Yaku[]
```

### Deliverable

* List of yaku + han for a given hand structure

### Acceptance test

* Correct yaku list for known hands

### Do NOT:

* Add yakuman
* Add fu-dependent yaku

---

## STEP 7 — Best interpretation selection

### Goal

Resolve ambiguous hands correctly.

### Tasks

1. Score each HandStructure:

   * sum han
   * add riichi / tsumo
2. Select max han result

### Deliverable

```ts
scoreHand(structures, context): BestResult
```

### Acceptance test

* Chiitoitsu vs pinfu resolved correctly
* Iipeikou detected when applicable

### Do NOT:

* Break ties by fu
* Optimize prematurely

---

## STEP 8 — Dora (MVP edition)

### Goal

Add dora without complicating logic.

### Tasks

1. Accept one dora indicator
2. Compute actual dora tile
3. Count occurrences in hand

### Deliverable

```ts
countDora(tiles, doraIndicator): number
```

### Acceptance test

* Dora counted correctly

### Do NOT:

* Add ura dora
* Add kan dora

---

## STEP 9 — End-to-end CLI or function call

### Goal

Prove the MVP works as a system.

### Tasks

* Single entry point:

```ts
evaluateHand(input): Result
```

Returns:

* isWinning
* best yaku list
* total han

### Deliverable

* CLI or simple function call
* Text output is fine

### Do NOT:

* Build UI
* Polish formatting

---

## STEP 10 — Regression tests (non-negotiable)

### Goal

Prevent rule regressions.

### Tasks

* Create a test file with:

  * 10 standard hands
  * 5 ambiguous hands
  * 5 invalid hands

### Deliverable

* Passing test suite

---

# STOP POINT (IMPORTANT)

At this point:
✅ unordered input works
✅ ambiguous hands resolved
✅ yaku detected
✅ dora counted

**This is a legitimate riichi engine.**

Everything else (fu, open hands, yakuman, UI) is **post-MVP**.

---

Great choice. Rust is *very* well-suited to this problem (enums, pattern matching, immutability, test discipline), and this kind of rules engine is a fantastic way to really learn it.

Below is a **drop-in section** you can paste directly into your guide.

---

## Rust Project Scaffolding (MVP)

### Goals of the scaffold

* Keep **game logic pure** and testable
* Make illegal states hard to represent
* Avoid lifetime/borrow hell early
* Allow incremental expansion (fu, open hands, UI later)

This is a **library-first** design with an optional CLI wrapper.

---

## 1. Create the project

```bash
cargo new riichi_engine --lib
cd riichi_engine
```

Why `--lib`:

* The core engine should be reusable
* CLI / web UI can be added later without refactoring

---

## 2. Top-level directory layout (MVP)

```text
riichi_engine/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── tile.rs
│   ├── parse.rs
│   ├── hand.rs
│   ├── decompose.rs
│   ├── wait.rs
│   ├── context.rs
│   ├── yaku/
│   │   ├── mod.rs
│   │   ├── basic.rs
│   │   └── structural.rs
│   ├── score.rs
│   └── dora.rs
└── tests/
    └── integration.rs
```

Each module has **one clear responsibility**.

---

## 3. Core data modeling (very important in Rust)

### `tile.rs`

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum Suit {
    Man,
    Pin,
    Sou,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum Honor {
    East,
    South,
    West,
    North,
    White,
    Green,
    Red,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum Tile {
    Suited { suit: Suit, value: u8 }, // 1–9
    Honor(Honor),
}
```

**Design notes**

* Use enums, not strings
* `Copy` avoids borrow headaches
* Illegal tiles cannot exist

---

### Tile counts (recommended pattern)

```rust
use std::collections::HashMap;

pub type TileCounts = HashMap<Tile, u8>;
```

Yes, an array would be faster — **don’t optimize yet**. HashMap is fine for MVP and clearer to reason about.

---

## 4. Parsing & validation layer

### `parse.rs`

Responsibilities:

* Convert user input → `Vec<Tile>`
* Validate:

  * 14 tiles
  * no tile > 4
  * legal symbols

Example API:

```rust
pub fn parse_tiles(input: &[&str]) -> Result<Vec<Tile>, String> { ... }

pub fn to_counts(tiles: &[Tile]) -> TileCounts { ... }
```

**Rust tip**

* Return `Result<_, String>` for MVP
* Use `thiserror` later if you want polish

---

## 5. Hand & meld representation

### `hand.rs`

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Meld {
    Sequence(Tile, Tile, Tile),
    Triplet(Tile),
}

#[derive(Debug, Clone)]
pub struct HandStructure {
    pub melds: Vec<Meld>,
    pub pair: Tile,
}
```

Design rule:

> A `HandStructure` is always **logically complete**.

---

## 6. Decomposition engine

### `decompose.rs`

Responsibilities:

* Given `TileCounts`, return **all** valid `HandStructure`s

Public API:

```rust
pub fn decompose(counts: &TileCounts) -> Vec<HandStructure>;
```

Implementation notes:

* Recursive, depth-first
* Clone `TileCounts` aggressively (simpler, safe)
* Performance is fine at riichi scale

Rust tip:

* Favor clarity over borrow cleverness here

---

## 7. Wait type detection

### `wait.rs`

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum WaitType {
    Ryanmen,
    Kanchan,
    Penchan,
    Tanki,
    Shanpon,
}
```

API:

```rust
pub fn detect_wait(
    hand: &HandStructure,
    winning_tile: Tile,
) -> WaitType;
```

This module should be **pure logic** with no scoring.

---

## 8. Game context

### `context.rs`

```rust
#[derive(Debug, Clone, Copy)]
pub enum WinType {
    Ron,
    Tsumo,
}

#[derive(Debug, Clone, Copy)]
pub enum Wind {
    East,
    South,
    West,
    North,
}

pub struct Context {
    pub win_type: WinType,
    pub seat_wind: Wind,
    pub round_wind: Wind,
    pub riichi: bool,
}
```

Rust win:

* Context is immutable
* Easy to extend later

---

## 9. Yaku detection modules

### `yaku/mod.rs`

```rust
pub mod basic;
pub mod structural;
```

### `yaku/basic.rs`

Context-dependent yaku:

* Riichi
* Menzen Tsumo
* Yakuhai

### `yaku/structural.rs`

Hand-structure yaku:

* Pinfu
* Tanyao
* Iipeikou
* Toitoi
* Chiitoitsu

Common API:

```rust
pub fn detect_yaku(
    hand: &HandStructure,
    ctx: &Context,
) -> Vec<Yaku>;
```

---

## 10. Scoring & best-hand selection

### `score.rs`

```rust
pub struct ScoreResult {
    pub yaku: Vec<Yaku>,
    pub han: u32,
}

pub fn score_best(
    hands: &[HandStructure],
    ctx: &Context,
) -> Option<ScoreResult>;
```

Responsibilities:

* Evaluate all interpretations
* Select highest han

---

## 11. Dora counting

### `dora.rs`

```rust
pub fn count_dora(
    tiles: &[Tile],
    indicator: Tile,
) -> u32;
```

Keep this isolated — it plugs in cleanly later.

---

## 12. `lib.rs` (public API surface)

```rust
pub mod tile;
pub mod parse;
pub mod hand;
pub mod decompose;
pub mod wait;
pub mod context;
pub mod yaku;
pub mod score;
pub mod dora;

pub fn evaluate_hand(...) -> ... { ... }
```

Rule:

> `lib.rs` should read like a table of contents.

---

## 13. Tests (critical for Rust learning)

### Unit tests

* Put next to each module
* Test small logic units

### Integration tests

`tests/integration.rs`

* End-to-end hand evaluations

Rust tip:

> Write tests *before* refactoring. Rust will force correctness.

---

## Rust-specific advice for this project

* Prefer `Clone` over fighting the borrow checker early
* Pattern matching on enums will be your superpower
* Keep functions small and pure
* Use `#[derive(Debug)]` everywhere at first
