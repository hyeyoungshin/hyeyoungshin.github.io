---
layout: post
title: "immutable data structure and mutable binding"
date: 2026-02-10
---

I was a bit confused why there are examples of immutable data structures that are declared `mut` and used as a mutable data. 

For example, I was looking at the docs for im::Vector and ran into this example

```rust
let mut vec = vector![5, 6, 7];
vec.push_front(4);

assert_eq!(vector![4, 5, 6, 7], vec);
```

The distiction that needs to be made is that in this example `vec` is indeed an **immutable data structure** but with a **mutable binding**, which is a separate concept and does not make the immutable data structure mutable. Instead what it does is keeping the original data in tact behind the scenes and make it _appear_ to be mutated.


Here is what's actually happening:
```rust
let mut vec = vector![5, 6, 7];
vec = vec.push_front(4);  // push_front returns NEW vec, reassign to same binding
``` 
`push_front` does not mutate the original vector. It returns a new vector with structural sharing, then reassigns it to the same variable.

### Why This Design?
The `im` crate provides both APIs for ergonimics.

**API 1: Functional (returns new value)**
```rust
let vec1 = vector![5, 6, 7];
let vec2 = vec1.push_front(4);  // Returns new vector
// vec1 is still [5, 6, 7]
// vec2 is [4, 5, 6, 7]
```

**API 2: "Mutation" (reassignment syntactic sugar)**
```rust
let mut vec = vector![5, 6, 7];
vec.push_front(4);  // Internally: vec = vec.push_front(4)
// vec is now [4, 5, 6, 7]
```

Both do the same thing: create a new vector. The second API is just more convenient.

### Why Use `im::vector`?
The benefit of immutable vectors is not the `mut` binding, but is keeping old versions. It might be a good data structure to represent my `turn_order` where dynamic player join/leave mid-game is allowed. 

To summarize, the "mutability" of `im::vector` was my misunderstanding. `let mut vec` is mutable binding but the data structure itself is not mutable. It creates new versions each time. 
