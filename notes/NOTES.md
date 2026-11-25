# MESSY NOTES

## Parsing

- String to number: `<string slice>.parse::<u64>().unwrap()`
  - `parse` uses trait `FromStr` function `from_str()` that can be overridden
- Define a struct to use `struct Pos {x: i64, y: i64}`

## Processing

```rust
input
  .split("\n\n")  // Split on two newlines
  .map(|batch| {  // For each section
    batch
      .lines()
      .map(|line| line.parse::<u64>().unwrap())  // For each line in section
      .sum::<u64>()
  })
  .max()  // For each section get max
  .unwrap()

  // End could be this
  .sorted() // from itertools
  .rev()
  .take(3)
  .sum()
```

```rust
// Return a vector of vectors
// Parse each line to be a vector of i32 numbers
fn parse_new_lines_into_vectors(input: String) -> Vec<Vec<i32>> {
    let out: Vec<_> = input
        .lines()
        .map(|line| {
            line.chars()
                .filter_map(|c| c.to_digit(10))
                .map(|d| d as i32)
                .collect()
        })
        .collect();

    out
}
```

- `itertools`
- `regex`
  - Can use named capture groups to later refernce

```rust
  let re = Regex::new(r"<SOME REGEX> x=(?P<x1>[-]?\d+), .....").unwrap();
  let caputures = re.captures(line).unwrap();
  let my_point = Pos {
    x: captures["x1"].parse().unwrap(),
    y: captures["y1"].parse().unwrap(),
  }
```

```rust
  fn from_str(s: &str) -> Result<Self, Self::Err> {...}
```

### Debugging iterators:

```rust
    .inspect(|x| println!("Value: {:?}", x))
    .map(|x| dbg!(x))
```

### `nom` Example:

- `nom` - <https://tfpk.github.io/nominomicon/chapter_1.html>

```rust
use nom::{
    IResult, Parser,
    bytes::complete::tag,
    character::complete::{alpha1, anychar, char, line_ending, u32},
    combinator::all_consuming,
    multi::separated_list1,
    sequence::separated_pair,
};

// The typed result we want out of each line.
#[derive(Debug)]
struct Policy {
    min: u32,
    max: u32,
    letter: char,
    password: String,
}

// Parses a single line like "3-7 a: abcde".
fn parse_line(input: &str) -> IResult<&str, Policy> {
    // A tuple of parsers runs each one in order,
    // feeding the leftover input to the next.
    // The result is a tuple of every parser's output.
    let (rest, ((min, max), _, letter, _, password)) = (
        separated_pair(u32, char('-'), u32), // "3-7"  -> (3, 7)
        char(' '),                           // the space (value ignored)
        anychar,                             // "a"
        tag(": "),                           // literal ": " (ignored)
        alpha1,                              // "abcde" (one or more letters)
    )
        .parse(input)?;

    // Build the struct from the pieces we care about.
    let policy = Policy {
        min,
        max,
        letter,
        password: password.to_string(),
    };

    Ok((rest, policy))
}

// Parses the whole file: one policy per line.
fn parse_input(input: &str) -> IResult<&str, Vec<Policy>> {
    separated_list1(line_ending, parse_line).parse(input)
}

fn main() {
    const INPUT: &str = include_str!("../input/day02.txt");

    // all_consuming fails if any input is left over, which catches
    // a parser that quietly stopped halfway through the file.
    // trim() removes the trailing newline at the end of the file.
    let (_, policies) = all_consuming(parse_input)
        .parse(INPUT.trim())
        .expect("failed to parse input");

    println!("Parsed {} policies", policies.len());
}
```

## Review

- Option
- Result
- HashMap
- Vec
- `itertools`
- `regex`
- `nom` - <https://tfpk.github.io/nominomicon/chapter_1.html>
- Review of 2023 - <https://www.youtube.com/playlist?list=PLWtPciJ1UMuD3_8Pb-EqrFhkYpastR2cn>
