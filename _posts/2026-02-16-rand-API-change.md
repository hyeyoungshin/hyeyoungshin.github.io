---
layout: post
title: "rand API change in 0.10"
date: 2026-02-16
---

The `rand` library's API changed slightly in its latest version 0.10. This was part of a broader cleanup. 

### The old way of generating a random number (`rand` <= **0.8**):

```rust
use rand::Rng;

let mut rng = rand::thread_rng();
let n = rng.gen_range(0..10);
```

### The new way (`rand` **0.10**):

```rust
use rand::prelude::*;

let mut rng = rand::rng();
let n = rng.random_range(0..10);
```

Just renaming, no semantic changes...

**A little more explanation**

The `rand::Rng` trait moved under the **prelude** module and is no longer re-exported at the root the same as before. 

In earlier versions like 0.8 and 0.9, the `Rng` trait was commonly imported directly from the crate root. However, in `rand` 0.10, the crate recorganized its exports so that ,any commonly used traits live in `rand::prelude`. This is similar to how `std::prelude` works. 
