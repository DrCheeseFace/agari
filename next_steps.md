## Summary of Progress

We've completed **Steps 0-2** of your Riichi scoring engine MVP:

| Step | Status | What we built |
|------|--------|---------------|
| 0 | ✅ | Project setup, `RULES.md` |
| 1 | ✅ | Tile representation, parsing, validation |
| 2 | ✅ | Hand completeness checker (standard + chiitoitsu) |

**Tests passing:** 15

---

## Current File Structure

```
agari/
├── Cargo.toml
├── RULES.md
└── src/
    ├── lib.rs
    ├── tile.rs
    ├── parse.rs
    └── hand.rs
```

---

## Instructions for Resuming

Paste this into a new Claude chat:

---

**I'm building a Rust-based Riichi Mahjong hand scoring engine called "Agari". This is my first Rust project (I know Python). I'm working with a tutor/coach approach.**

**Completed so far (Steps 0-2):**
- Tile representation with `Suit`, `Honor`, `Tile` enums
- Parser for notation like `123m456p789s11222z`
- Hand validation (14 tiles, max 4 copies)
- Completeness checker (standard 4 melds + pair, and chiitoitsu)

**Next up (Step 3):** Hand decomposition engine — generate ALL valid `HandStructure`s (not just check if valid)

**Current files:**

`Cargo.toml`:
```toml
[package]
name = "agari"
version = "0.1.0"
edition = "2024"
```

`src/lib.rs`:
```rust
pub mod tile;
pub mod parse;
pub mod hand;
```

`src/tile.rs`:
```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub enum Suit {
    Man,
    Pin,
    Sou,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub enum Honor {
    East,
    South,
    West,
    North,
    White,
    Green,
    Red,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub enum Tile {
    Suited { suit: Suit, value: u8 },
    Honor(Honor),
}

impl Tile {
    pub fn suited(suit: Suit, value: u8) -> Self {
        Tile::Suited { suit, value }
    }

    pub fn honor(honor: Honor) -> Self {
        Tile::Honor(honor)
    }

    pub fn is_simple(&self) -> bool {
        match self {
            Tile::Suited { value, .. } => *value >= 2 && *value <= 8,
            Tile::Honor(_) => false,
        }
    }

    pub fn is_terminal_or_honor(&self) -> bool {
        match self {
            Tile::Suited { value, .. } => *value == 1 || *value == 9,
            Tile::Honor(_) => true,
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn create_tiles() {
        let five_man = Tile::suited(Suit::Man, 5);
        let red_dragon = Tile::honor(Honor::Red);

        println!("{:?}", five_man);
        println!("{:?}", red_dragon);

        assert_eq!(five_man, Tile::suited(Suit::Man, 5));
    }

    #[test]
    fn tile_properties() {
        assert!(Tile::suited(Suit::Pin, 5).is_simple());
        assert!(!Tile::suited(Suit::Pin, 1).is_simple());
        assert!(!Tile::suited(Suit::Pin, 9).is_simple());
        assert!(!Tile::honor(Honor::East).is_simple());

        assert!(Tile::suited(Suit::Sou, 1).is_terminal_or_honor());
        assert!(Tile::suited(Suit::Sou, 9).is_terminal_or_honor());
        assert!(Tile::honor(Honor::White).is_terminal_or_honor());
        assert!(!Tile::suited(Suit::Man, 5).is_terminal_or_honor());
    }
}
```

`src/parse.rs`:
```rust
use std::collections::HashMap;
use crate::tile::{Tile, Suit, Honor};

pub type TileCounts = HashMap<Tile, u8>;

pub fn parse_hand(input: &str) -> Result<Vec<Tile>, String> {
    let mut tiles = Vec::new();
    let mut pending_numbers: Vec<u8> = Vec::new();

    for ch in input.chars() {
        match ch {
            '1'..='9' => {
                let digit = ch.to_digit(10).unwrap() as u8;
                pending_numbers.push(digit);
            }

            'm' => {
                for &n in &pending_numbers {
                    tiles.push(Tile::suited(Suit::Man, n));
                }
                pending_numbers.clear();
            }
            'p' => {
                for &n in &pending_numbers {
                    tiles.push(Tile::suited(Suit::Pin, n));
                }
                pending_numbers.clear();
            }
            's' => {
                for &n in &pending_numbers {
                    tiles.push(Tile::suited(Suit::Sou, n));
                }
                pending_numbers.clear();
            }

            'z' => {
                for &n in &pending_numbers {
                    let honor = match n {
                        1 => Honor::East,
                        2 => Honor::South,
                        3 => Honor::West,
                        4 => Honor::North,
                        5 => Honor::White,
                        6 => Honor::Green,
                        7 => Honor::Red,
                        _ => return Err(format!("Invalid honor number: {}", n)),
                    };
                    tiles.push(Tile::honor(honor));
                }
                pending_numbers.clear();
            }

            ' ' | '\t' | '\n' => {}

            _ => return Err(format!("Unexpected character: {}", ch)),
        }
    }

    if !pending_numbers.is_empty() {
        return Err("Trailing numbers without suit suffix".to_string());
    }

    Ok(tiles)
}

pub fn to_counts(tiles: &[Tile]) -> TileCounts {
    let mut counts = HashMap::new();
    for &tile in tiles {
        *counts.entry(tile).or_insert(0) += 1;
    }
    counts
}

pub fn validate_hand(tiles: &[Tile]) -> Result<(), String> {
    if tiles.len() != 14 {
        return Err(format!("Hand must have 14 tiles, got {}", tiles.len()));
    }

    let counts = to_counts(tiles);
    for (tile, count) in &counts {
        if *count > 4 {
            return Err(format!("Tile {:?} appears {} times (max 4)", tile, count));
        }
    }

    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parse_simple_hand() {
        let tiles = parse_hand("123m456p789s11z").unwrap();
        assert_eq!(tiles.len(), 11);
        assert_eq!(tiles[0], Tile::suited(Suit::Man, 1));
        assert_eq!(tiles[9], Tile::honor(Honor::East));
    }

    #[test]
    fn parse_full_hand() {
        let tiles = parse_hand("123m456p789s11222z").unwrap();
        assert_eq!(tiles.len(), 14);
    }

    #[test]
    fn parse_invalid_honor() {
        let result = parse_hand("89z");
        assert!(result.is_err());
    }

    #[test]
    fn parse_trailing_numbers() {
        let result = parse_hand("123");
        assert!(result.is_err());
    }

    #[test]
    fn validate_correct_hand() {
        let tiles = parse_hand("123m456p789s11222z").unwrap();
        assert!(validate_hand(&tiles).is_ok());
    }

    #[test]
    fn validate_wrong_count() {
        let tiles = parse_hand("123m456p789s11z").unwrap();
        assert!(validate_hand(&tiles).is_err());
    }

    #[test]
    fn validate_too_many_copies() {
        let tiles = parse_hand("11111m456p789s11z").unwrap();
        assert!(validate_hand(&tiles).is_err());
    }
}
```

`src/hand.rs`:
```rust
use crate::tile::Tile;
use crate::parse::TileCounts;

pub fn is_chiitoitsu(counts: &TileCounts) -> bool {
    counts.len() == 7 && counts.values().all(|&c| c == 2)
}

pub fn is_standard_hand(counts: &TileCounts) -> bool {
    for (&tile, &count) in counts {
        if count >= 2 {
            let mut remaining = counts.clone();
            *remaining.get_mut(&tile).unwrap() -= 2;
            
            if remaining[&tile] == 0 {
                remaining.remove(&tile);
            }
            
            if can_form_melds(remaining, 4) {
                return true;
            }
        }
    }
    false
}

fn can_form_melds(mut counts: TileCounts, needed: u32) -> bool {
    counts.retain(|_, &mut c| c > 0);
    
    if needed == 0 {
        return counts.is_empty();
    }
    
    if counts.is_empty() {
        return false;
    }
    
    let tile = *counts.keys().min().unwrap();
    let count = counts[&tile];
    
    if count >= 3 {
        let mut after_triplet = counts.clone();
        *after_triplet.get_mut(&tile).unwrap() -= 3;
        if can_form_melds(after_triplet, needed - 1) {
            return true;
        }
    }
    
    if let Tile::Suited { suit, value } = tile {
        if value <= 7 {
            let next1 = Tile::suited(suit, value + 1);
            let next2 = Tile::suited(suit, value + 2);
            
            let has_seq = counts.get(&next1).copied().unwrap_or(0) >= 1
                       && counts.get(&next2).copied().unwrap_or(0) >= 1;
            
            if has_seq {
                let mut after_seq = counts.clone();
                *after_seq.get_mut(&tile).unwrap() -= 1;
                *after_seq.get_mut(&next1).unwrap() -= 1;
                *after_seq.get_mut(&next2).unwrap() -= 1;
                if can_form_melds(after_seq, needed - 1) {
                    return true;
                }
            }
        }
    }
    
    false
}

pub fn is_winning_hand(counts: &TileCounts) -> bool {
    is_chiitoitsu(counts) || is_standard_hand(counts)
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::parse::{parse_hand, to_counts};

    #[test]
    fn test_chiitoitsu() {
        let tiles = parse_hand("1122m3344p5566s77z").unwrap();
        let counts = to_counts(&tiles);
        assert!(is_chiitoitsu(&counts));
        assert!(is_winning_hand(&counts));
    }

    #[test]
    fn test_not_chiitoitsu_four_of_kind() {
        let tiles = parse_hand("1111m22m33p44p55s66s").unwrap();
        let counts = to_counts(&tiles);
        assert!(!is_chiitoitsu(&counts));
    }

    #[test]
    fn test_standard_hand() {
        let tiles = parse_hand("123m456p789s11122z").unwrap();
        let counts = to_counts(&tiles);
        assert!(is_standard_hand(&counts));
        assert!(is_winning_hand(&counts));
    }

    #[test]
    fn test_all_triplets() {
        let tiles = parse_hand("111m222p333s44455z").unwrap();
        let counts = to_counts(&tiles);
        assert!(is_standard_hand(&counts));
    }

    #[test]
    fn test_invalid_hand() {
        let tiles = parse_hand("1234m5678p9s123z").unwrap();
        let counts = to_counts(&tiles);
        assert!(!is_winning_hand(&counts));
    }

    #[test]
    fn test_pinfu_shape() {
        let tiles = parse_hand("123456m789p234s55z").unwrap();
        let counts = to_counts(&tiles);
        assert!(is_standard_hand(&counts));
    }
}
```

**MVP Plan reference:** (attach the original RIICHI SCORING TOOL document)

**Please continue coaching me through Step 3: Hand decomposition engine.**

---

Good luck with the rest of the project! You're off to a solid start. 🀄