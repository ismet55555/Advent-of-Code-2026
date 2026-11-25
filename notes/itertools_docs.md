# `itertools`: A Short Guide

## Overview

itertools adds extra methods to every Rust iterator, covering the patterns
that puzzles ask for again and again: pairs, windows, combinations,
counting and grouping.

Importing one trait is all it takes. After that, you call the new methods
the same way you call `map` or `filter`.

Use it when:

- you need adjacent pairs or sliding windows
- you need every pair, combination or permutation of some items
- you are counting, grouping or de-duplicating items
- you want a one-line join, sort, or min/max with its position

Plain `map`, `filter` and `sum` chains do not need it.

## Setup

Add the crate to `Cargo.toml`, then import the `Itertools` trait. Without
the import, the new methods fail with "no method found" errors.

```toml
[dependencies]
itertools = "0.14"
```

```rust
use itertools::Itertools;
```

## Examples

Each example is a pattern.

### 1. Adjacent pairs

Counting how many depths are larger than the one before:

```rust
let depths = vec![199, 200, 208, 210, 200, 207];

let increases = depths
    .iter()
    // Yields (199, 200), (200, 208), (208, 210), ...
    // The ::<(_, _)> says each window holds two items.
    .tuple_windows::<(_, _)>()
    .filter(|(before, after)| after > before)
    .count();
```

### 2. Every pair

Finding the two numbers that add up to 2020, then multiplying them:

```rust
let answer = numbers
    .iter()
    // Every unordered pair, as a Vec of two references.
    .combinations(2)
    .find(|pair| pair[0] + pair[1] == 2020)
    .map(|pair| pair[0] * pair[1]);
```

### 3. Counting

Counting how often each letter appears, then finding the most common:

```rust
// counts() returns a HashMap<char, usize>.
let counts = "abracadabra".chars().counts();

// Each entry is (&char, &usize), so compare on the count.
let most_common = counts
    .iter()
    .max_by_key(|(_, count)| **count);
```

### 4. Groups split by blank lines

```rust
let mut groups: Vec<Vec<&str>> = Vec::new();

// chunk_by puts consecutive lines with the same key in one chunk.
// Here the key is "is this line blank?".
for (is_blank, lines) in &input.lines().chunk_by(|line| line.is_empty()) {
    // Skip the chunks made of blank separator lines.
    if !is_blank {
        groups.push(lines.collect());
    }
}
```

## Tips

Annotate sizes, watch output sizes, and prefer tuples to `Vec`s.

- **Annotate tuple sizes.** `tuple_windows::<(_, _)>()` tells Rust how big
  each window is. Without it, inference often fails.
- **Prefer tuples over `Vec`s for pairs.** `combinations(2)` allocates a
  `Vec` per pair. `tuple_combinations::<(_, _)>()` yields plain tuples.
- **Watch the size of the output.** `permutations(k)` grows factorially:
  10 items taken 10 at a time is 3,628,800 orderings.
- **Know `dedup` vs `unique`.** `dedup` removes only neighbors that repeat.
  `unique` removes every repeat but needs `Hash` and `Eq`.
- **Sort `counts()` before printing or comparing.** `counts()` returns a
  `HashMap`, whose order changes between runs.
- **Use `collect_vec()`.** It replaces `collect::<Vec<_>>()` everywhere.
- **Use `izip!` for three or more zips.** It yields flat tuples like
  `(x, y, z)` instead of `((x, y), z)`.
- **Use `cartesian_product` for grid coordinates.** It replaces two nested
  `for` loops with one.

## Cheat sheet

The methods that cover most puzzles, grouped by what they do. All are
methods on an iterator unless the name ends in `!`.

### Pairs and windows

| Method | What it does |
| --- | --- |
| `tuple_windows::<(_, _)>()` | Sliding window of adjacent items |
| `tuple_combinations::<(_, _)>()` | Every unordered pair, as tuples |
| `circular_tuple_windows()` | Adjacent pairs, wrapping around |
| `tuples::<(_, _)>()` | Non-overlapping groups, as tuples |
| `zip_eq(other)` | Zip, panics if lengths differ |
| `interleave(other)` | Alternates items from two iterators |

### Combinatorics

| Method | What it does |
| --- | --- |
| `combinations(k)` | All k-sized subsets, as `Vec`s |
| `permutations(k)` | All k-sized orderings, as `Vec`s |
| `combinations_with_replacement(k)` | k-subsets that allow repeats |
| `powerset()` | Every subset, as `Vec`s |
| `cartesian_product(other)` | Every pairing of two iterators |
| `iproduct!(a, b, c)` | Cartesian product of many |
| `multi_cartesian_product()` | Product of an iterator of iterators |

### Counting and grouping

| Method | What it does |
| --- | --- |
| `counts()` | `HashMap` of item to times seen |
| `counts_by(f)` | Same, counting by a derived key |
| `into_group_map()` | Turns `(k, v)` pairs into `HashMap<k, Vec<v>>` |
| `into_group_map_by(f)` | Groups items by a computed key |
| `chunk_by(f)` | Groups runs of consecutive equal keys |
| `coalesce(f)` | Merges neighbors using a closure |

### Duplicates

| Method | What it does |
| --- | --- |
| `dedup()` | Removes consecutive duplicates |
| `unique()` | Removes all duplicates, keeps the first |
| `duplicates()` | Yields items seen more than once |
| `all_equal()` | True if every item is equal |
| `all_unique()` | True if no item repeats |

### Sorting

| Method | What it does |
| --- | --- |
| `sorted()` | Sorted items, as an iterator |
| `sorted_by_key(f)` | Sorts by a computed key |
| `sorted_by(f)` | Sorts with a comparison closure |
| `sorted_unstable()` | Faster sort when ties do not matter |

### Min, max and search

| Method | What it does |
| --- | --- |
| `minmax()` | Min and max in one pass |
| `position_max()`, `position_min()` | Index of the max or min |
| `max_set()`, `min_set()` | All items tied for max or min |
| `find_position(f)` | `(index, item)` of the first match |
| `positions(f)` | Every index where a test holds |

### Joining and collecting

| Method | What it does |
| --- | --- |
| `join(sep)` | Items into one `String`, split by `sep` |
| `collect_vec()` | Shortcut for `collect::<Vec<_>>()` |
| `collect_tuple::<(_, _)>()` | Exactly N items into a tuple, or `None` |
| `exactly_one()` | The only item, or an error |
| `partition_map(f)` | Splits into Left and Right collections |

### Building iterators

| Function | What it does |
| --- | --- |
| `izip!(a, b, c)` | Zips any number of iterators |
| `iterate(start, f)` | Infinite `start`, `f(start)`, `f(f(start))`, ... |

### Quick examples

```rust
(0..height).cartesian_product(0..width)   // every (row, col) cell
[3, 1, 2].iter().sorted().join(",")       // "1,2,3"
v.iter().tuple_windows::<(_, _, _)>()     // windows of three
v.iter().permutations(3)                  // all 3-item orderings
v.iter().minmax().into_option()           // Some((min, max))
iterate(1, |x| x * 2).take(5)             // 1, 2, 4, 8, 16
```
