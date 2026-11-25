# `std::collections`: A Short Guide

## Overview

The standard library already has every container a puzzle needs. No extra
crate is required, only an import.

Picking the right one is most of the work:

| Need | Use |
| --- | --- |
| Count things or look up by key | `HashMap` |
| Ask "have I seen this before?" | `HashSet` |
| Keys in sorted order, or range queries | `BTreeMap`, `BTreeSet` |
| First in, first out queue (BFS) | `VecDeque` |
| Always take the smallest or largest | `BinaryHeap` |
| Last in, first out stack | `Vec` with `push` and `pop` |

## Setup

Import only the types you use. `Reverse` lives in `std::cmp` and turns a
`BinaryHeap` into a min-heap.

```rust
use std::cmp::Reverse;
use std::collections::{
    BTreeMap, BTreeSet, BinaryHeap, HashMap, HashSet, VecDeque,
};
```

## Examples

Each example is a pattern.

### 1. Counting and grouping with `HashMap`

```rust
let mut counts: HashMap<char, usize> = HashMap::new();

for ch in "hello world".chars() {
    // entry() finds the slot for this key.
    // or_insert(0) creates it as 0 if it is missing.
    // The * lets us add to the value in place.
    *counts.entry(ch).or_insert(0) += 1;
}

// Group values by key. or_default() starts an empty Vec on first use.
let mut groups: HashMap<u32, Vec<&str>> = HashMap::new();
groups.entry(3).or_default().push("three");
groups.entry(3).or_default().push("tres");
```

### 2. Tracking visited items with `HashSet`

```rust
let mut seen: HashSet<(i32, i32)> = HashSet::new();

// insert() returns true only when the value is new,
// so one call both checks and records.
if seen.insert((2, 5)) {
    println!("first visit");
}

// Set operations return iterators. Collect to build a new set.
let a: HashSet<i32> = [1, 2, 3].into();
let b: HashSet<i32> = [2, 3, 4].into();
let both: HashSet<i32> = a.intersection(&b).copied().collect();
```

### 3. Breadth-first search with `VecDeque`

```rust
let mut seen = HashSet::new();
let mut queue = VecDeque::new();

seen.insert(start);
queue.push_back((start, 0));

// pop_front takes the oldest item, so we explore level by level.
while let Some((pos, steps)) = queue.pop_front() {
    // neighbors() is your puzzle-specific function.
    for next in neighbors(pos) {
        if seen.insert(next) {
            queue.push_back((next, steps + 1));
        }
    }
}
```

### 4. Cheapest-first search with `BinaryHeap`

```rust
let mut heap = BinaryHeap::new();

// BinaryHeap pops the LARGEST item. Reverse flips that,
// so the smallest cost comes out first.
// Tuples compare left to right, so put the cost first.
heap.push(Reverse((0, start)));

while let Some(Reverse((cost, node))) = heap.pop() {
    // Explore neighbors, then push Reverse((new_cost, next)).
}
```

### 5. Sorted keys and ranges with `BTreeMap`

```rust
let mut by_time: BTreeMap<u32, &str> = BTreeMap::new();
by_time.insert(30, "c");
by_time.insert(10, "a");
by_time.insert(20, "b");

// Iterates in key order: 10, 20, 30.
for (time, name) in &by_time {
    println!("{time}: {name}");
}

// Only the entries with keys from 10 up to and including 20.
for (time, name) in by_time.range(10..=20) {
    println!("{time}: {name}");
}

// The smallest entry, as Some((&10, &"a")).
let first = by_time.first_key_value();
```

## Tips

Choose by need, avoid random order, and skip extra lookups.

- **Sort before you print.** `HashMap` and `HashSet` iterate in a random
  order that changes between runs. Collect and sort when order matters.
- **Use `entry` instead of `get` then `insert`.** It does one lookup and
  handles the missing-key case for you.
- **Let `insert` double as a visited check.** `HashSet::insert` returns
  `true` only for new values, so one call tests and records.
- **Use `VecDeque` for queues.** `Vec::remove(0)` shifts every item, which is
  slow. `pop_front` is constant time.
- **Remember `BinaryHeap` is a max-heap.** Wrap items in `Reverse` to pop the
  smallest first, and put the cost first in a tuple.
- **Use integers for keys.** Tuples like `(i32, i32)` work as keys. Floats do
  not implement `Eq`, `Hash` or `Ord`, so they cannot be keys.
- **Swap in a faster hasher for hot loops.** The default hasher is slow.
  `FxHashMap` from the `rustc-hash` crate is a drop-in replacement.
- **Use a `Vec` for dense grids.** A `Vec<Vec<T>>` indexed by row and column
  beats a `HashMap` when every cell is filled.
- **Pick `BTreeMap` for "next bigger" questions.** `range` finds the
  nearest key above or below a value without scanning.

## Cheat sheet

The methods that cover most puzzles. `HashMap` and `BTreeMap` share the
same core API, as do `HashSet` and `BTreeSet`.

### HashMap and BTreeMap

| Method | What it does |
| --- | --- |
| `insert(k, v)` | Adds or replaces, returns the old value |
| `get(&k)` | Returns `Option<&V>` |
| `get_mut(&k)` | Returns `Option<&mut V>` for in-place edits |
| `contains_key(&k)` | True if the key exists |
| `remove(&k)` | Removes the entry, returns `Option<V>` |
| `entry(k).or_insert(v)` | Gets the value, inserting `v` if absent |
| `entry(k).or_default()` | Same, inserting `V::default()` |
| `entry(k).and_modify(f).or_insert(v)` | Updates if present, else inserts |
| `keys()`, `values()` | Iterators over keys or values |
| `values_mut()` | Iterator over `&mut V` |
| `iter()` | Iterator over `(&K, &V)` pairs |
| `retain(f)` | Keeps only entries where `f` is true |
| `extend(iter)` | Adds many `(k, v)` pairs |
| `into_keys()`, `into_values()` | Consuming iterators |
| `len()`, `is_empty()` | Size checks |

### HashSet and BTreeSet

| Method | What it does |
| --- | --- |
| `insert(x)` | Adds `x`, returns true if it was new |
| `remove(&x)` | Removes `x`, returns true if it was there |
| `contains(&x)` | Membership test |
| `union(&other)` | Items in either set |
| `intersection(&other)` | Items in both sets |
| `difference(&other)` | Items in this set but not the other |
| `symmetric_difference(&other)` | Items in exactly one of the sets |
| `is_subset(&other)` | True if every item is in `other` |
| `is_disjoint(&other)` | True if the sets share nothing |

The set operations return iterators. Add `.copied().collect()` to build a
new set, as in `a.intersection(&b).copied().collect::<HashSet<_>>()`.

### VecDeque

| Method | What it does |
| --- | --- |
| `push_back(x)`, `push_front(x)` | Adds to either end |
| `pop_front()`, `pop_back()` | Removes from either end, returns `Option` |
| `front()`, `back()` | Peeks at either end |
| `iter()` | Front-to-back iterator |
| `rotate_left(n)` | Moves `n` items from the front to the back |
| `make_contiguous()` | Returns a slice, handy for sorting |

### BinaryHeap

| Method | What it does |
| --- | --- |
| `push(x)` | Adds an item |
| `pop()` | Removes the largest item, returns `Option` |
| `peek()` | Looks at the largest item without removing it |
| `into_sorted_vec()` | All items, smallest first |
| `Reverse(x)` | Wrapper that flips the order (min-heap) |

### BTreeMap and BTreeSet extras

| Method | What it does |
| --- | --- |
| `first_key_value()`, `last_key_value()` | Smallest or largest entry |
| `first()`, `last()` | Same for `BTreeSet` |
| `pop_first()`, `pop_last()` | Removes the smallest or largest entry |
| `range(a..b)` | Entries with keys inside the range |

### Quick examples

```rust
let map: HashMap<_, _> = [("a", 1), ("b", 2)].into();
let set: HashSet<_> = vec![1, 2, 2, 3].into_iter().collect(); // {1, 2, 3}
let heap: BinaryHeap<_> = [3, 1, 4].into();                   // max-heap
let queue: VecDeque<_> = (1..=3).collect();                   // 1, 2, 3
let sorted: BTreeSet<_> = [3, 1, 2].into();                   // 1, 2, 3
let word_lengths: HashMap<_, _> = ["hi", "hey"]
    .iter()
    .map(|w| (*w, w.len()))
    .collect();
```
