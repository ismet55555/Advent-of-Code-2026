# `nom`: A Short Guide

## Overview

nom builds parsers for puzzle input by snapping small functions together,
which beats `split` and `parse` once a format has alternatives, nesting or
repeats.

Every nom parser has the same shape: it takes the remaining input and
returns the leftover text plus the parsed value, or an error.

```rust
fn parser(input: &str) -> IResult<&str, Output>
```

Use nom when:

- a line mixes several kinds of pieces, like `Game 1: 3 blue, 4 red; 1 red`
- a value can take one of several forms
- the input has nested lists or repeated groups

Skip nom when a line is just numbers or `x=3,y=4`. Plain `split` and
`parse` take less code there.

## Setup

Add nom 8 to `Cargo.toml` and import from the `complete` modules, since
puzzle input is always fully in memory.

```toml
[dependencies]
nom = "8"
```

```rust
use nom::{
    IResult, Parser,
    branch::alt,
    bytes::complete::tag,
    character::complete::{alpha1, anychar, char, line_ending, u32},
    combinator::all_consuming,
    multi::separated_list1,
    sequence::{preceded, separated_pair},
};
```

## Examples

Each example builds on small parsers and snaps them together.

### 1. One line into a struct

Parsing `3-7 a: abcde`:

```rust
#[derive(Debug)]
struct Policy {
    min: u32,
    max: u32,
    letter: char,
    password: String,
}

fn parse_line(input: &str) -> IResult<&str, Policy> {
    // A tuple of parsers runs each one in order,
    // passing the leftover input along.
    // The output is a tuple of each parser's result.
    let (rest, ((min, max), _, letter, _, password)) = (
        separated_pair(u32, char('-'), u32), // "3-7"   -> (3, 7)
        char(' '),                           // space (ignored)
        anychar,                             // "a"
        tag(": "),                           // literal (ignored)
        alpha1,                              // "abcde"
    )
        .parse(input)?;

    let policy = Policy {
        min,
        max,
        letter,
        password: password.to_string(),
    };

    Ok((rest, policy))
}
```

### 2. Choosing between options

`alt` tries each parser in order and keeps the first that succeeds. Every
branch must return the same type.

```rust
#[derive(Debug, PartialEq)]
enum Color {
    Red,
    Green,
    Blue,
}

fn parse_color(input: &str) -> IResult<&str, Color> {
    alt((
        // tag() matches the text, .map() turns it into our enum.
        tag("red").map(|_| Color::Red),
        tag("green").map(|_| Color::Green),
        tag("blue").map(|_| Color::Blue),
    ))
    .parse(input)
}
```

### 3. Nested lists

Parsing `Game 1: 3 blue, 4 red; 1 red` by stacking one small parser per
level:

```rust
// "3 blue" -> (3, Color::Blue)
fn parse_cube(input: &str) -> IResult<&str, (u32, Color)> {
    separated_pair(u32, char(' '), parse_color).parse(input)
}

// "3 blue, 4 red" -> list of cubes
fn parse_hand(input: &str) -> IResult<&str, Vec<(u32, Color)>> {
    separated_list1(tag(", "), parse_cube).parse(input)
}

// "Game 1: <hand>; <hand>" -> (1, list of hands)
fn parse_game(input: &str) -> IResult<&str, (u32, Vec<Vec<(u32, Color)>>)> {
    separated_pair(
        preceded(tag("Game "), u32),            // "Game 1" -> 1
        tag(": "),
        separated_list1(tag("; "), parse_hand),  // hands split by "; "
    )
    .parse(input)
}
```

Running it over the whole file:

```rust
const INPUT: &str = include_str!("../input/day02.txt");

let (_, games) = all_consuming(separated_list1(line_ending, parse_game))
    .parse(INPUT.trim())
    .expect("failed to parse input");
```

## Tips

Build small, test each piece, and fail loudly at the top level.

- **Build bottom-up.** Write `parse_color` first, then `parse_cube`, then
  `parse_hand`. Each is a plain function, so a test is one line:
  `assert_eq!(parse_cube("3 blue"), Ok(("", (3, Color::Blue))));`
  (derive `PartialEq` and `Debug` on your types).
- **Wrap the top level in `all_consuming`.** Without it, a parser that
  stops halfway through the file still returns `Ok` and gives you a wrong
  answer.
- **Call `.trim()` on the input first.** The trailing newline at the end of
  the file otherwise makes `all_consuming` fail.
- **Import from `complete`, not `streaming`.** The streaming versions are
  for data that arrives in chunks.
- **Parse into typed structs and enums once.** Then `part1` and `part2`
  work on clean data and never touch strings.
- **Use signed types for negatives.** `i64` accepts a leading `-`, `u32`
  does not.
- **Read errors by their leftover text.** A failed parse reports the input
  remaining at the point of failure, which shows exactly where your parser
  stopped.
- **Use `.finish()` for a plain `Result`.** Import `nom::Finish` to turn
  nom's `Err` wrapper into a normal `Result`.
- **Keep `split` for easy lines.** Mixing both in one solution is fine.

## Cheat sheet

The parsers and combinators that cover most puzzle inputs, grouped by the
module they come from.

### Text: `bytes` and `character::complete`

| Parser | What it does |
| --- | --- |
| `tag("mul(")` | Matches an exact string |
| `char(',')` | Matches one specific character |
| `anychar` | Takes any one character |
| `alpha1` | One or more letters |
| `digit1` | One or more digits, returned as text |
| `alphanumeric1` | One or more letters or digits |
| `take_until(":")` | Everything before the first `:` |
| `take_while1(char::is_alphabetic)` | Chars while a test is true |

### Whitespace: `character::complete`

| Parser | What it does |
| --- | --- |
| `space0`, `space1` | Spaces and tabs (zero+ or one+) |
| `multispace0`, `multispace1` | Whitespace including newlines |
| `line_ending` | One newline, `\n` or `\r\n` |
| `not_line_ending` | The rest of the current line |

### Numbers: `character::complete` and `combinator`

| Parser | What it does |
| --- | --- |
| `u32`, `u64` | Unsigned integer |
| `i32`, `i64` | Signed integer, accepts a leading `-` |
| `map_res(digit1, str::parse::<usize>)` | Digits parsed into any type |

### Combining: `sequence`, `branch`, `combinator`

| Parser | What it does |
| --- | --- |
| `(a, b, c)` | Runs in order, returns a tuple |
| `preceded(a, b)` | Runs `a`, drops it, keeps `b` |
| `terminated(a, b)` | Keeps `a`, drops `b` |
| `delimited(a, b, c)` | Keeps only the middle one |
| `separated_pair(a, sep, b)` | Keeps `a` and `b`, drops `sep` |
| `alt((a, b))` | First success wins, same output type |
| `opt(p)` | Optional, returns an `Option` |
| `p.map(f)` | Transforms the parsed value |
| `value(v, p)` | Matches `p`, returns constant `v` |

### Repeating: `multi`

| Parser | What it does |
| --- | --- |
| `many0(p)`, `many1(p)` | Repeats `p`: zero+ or one+ |
| `separated_list1(sep, p)` | One or more items split by `sep` |
| `separated_list0(sep, p)` | Same, but an empty list is fine |
| `count(p, n)` | Exactly `n` repeats |
| `many_till(p, end)` | Repeats `p` until `end` matches |

### Finishing: `combinator` and `Finish`

| Parser | What it does |
| --- | --- |
| `all_consuming(p)` | Fails unless all input is used |
| `.finish()` | Turns nom's `Err` into a plain `Result` |

### Quick examples

```rust
preceded(tag("Game "), u32)           // "Game 12"  -> 12
delimited(char('('), u32, char(')'))  // "(7)"      -> 7
separated_list1(char(','), u32)       // "1,2,3"    -> [1, 2, 3]
separated_pair(u32, char('-'), u32)   // "3-7"      -> (3, 7)
alt((tag("on"), tag("off")))          // "on" or "off"
value(Color::Red, tag("red"))         // "red"      -> Color::Red
```
