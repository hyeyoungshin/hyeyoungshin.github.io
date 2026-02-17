---
layout: post
title: "rand API change in 0.10"
date: 2026-02-16
---

The `rand` library's API changed slightly in its latest version 0.10. 

**One of the changes**:  

The `rand::Rng` trait moved under the **prelude** module and is no longer re-exported at the root the same as before. So if I used it with the line 

```rust
use rand::Rng;
``` 
before, it should now be

```rust
use rand::prelude::*;

fn main() {
    let mut rng = rand::rng();
    let random_number = rng.random_range(0..10);
    //...
}
```

for `.random_range` method to work. 

In earlier versions like 0.8 and 0.9, the `Rng` trait was commonly imported directly from the crate root. However, in `rand` 0.10, the crate recorganized its exports so that ,any commonly used traits live in `rand::prelude`. This is similar to how `std::prelude` works. 
