---
layout: post
title: "adding syntax highlight is easy, but choosing a theme is not"
date: 2026-01-29
---

The following erroneous example made me wonder what Rust compiler knows and doesn't know. Of course, it comes down to the fundamental distinction between **compiler-time (static)** and **runtime (dynamic)** information.

```rust
let num: i32 = gen_range(0..2);

match num {  // num: i32
    0 => {},
    1 => {},
    // Not exhaustive! Compiler doesn't know num is either 0 or 1
    // It only knows num has type i32, so it could be any value from -2^31 to 2^31-1
}
```
Rust requires exhaustiveness based on what it can statically verify.


### What Rust's Type Checker Knows (Compile Time/Ststic)

1. Types

```rust
let x: i32 = some_function();
// what compiler knows: x's type
// what compiler does not know: the value of x
```

2. Type bounds and constraints 

```rust
fn process<T: Display>(item: T) {
    // What compiler knows: T implements Display
    // What compiler does not know: the concrete type of T at runtime
}
```

3. Struct/enum definitions

```rust
enum Animal {
    Dog,
    Cat,
}
// What compiler knows: Animal can only be Dog or Cat
// This is encoded in the type system
``` 

4. Lifetimes

```rust
fn foo<'a>(x: &'a str) -> &'a str
// What compiler knows: returned reference lives as long as input reference
```

5. Control flow structure

```rust
match value {
    0 => ...,
    1 => ...,
    // What compiler knows: these are the branches
    // What compiler does not know: which branch will execute
}
```

### What Rust's Type Checker Doesn't Know (Runtime/Dynamic)

1. Actual values

```rust
let x = rand::thread_rng().get_range(0..2);
// What compiler knows: x is i32
// What compiler does not know: x will be 0 or 1 --- runtime fact
``` 

2. Function return values

```rust
fn get_number() -> i32 {
    ...
}

let n = get_number()l
// What compiler knows: n is i32
// What compiler does not know: the value of n
```

3. Runtime conditions

```rust
if some_condition() { // Evaluated at runtime
    // What compiler does not know: whether this branch runs
}
```

4. Contents of collections 

```rust
let vec = vec![1, 2, 3];
// What compiler knows: vec is Vec<i32>
// What compiler does not know: vec contains [1, 2, 3] or even how many elements
``` 

***The Key Principle: Type System vs Value Space**


Going back to the erroneous example, what is a good way to fix the error?

1.  Use a type that encodes the constraint

```rust
// Instead of i32 with runtime range [0, 2)
enum BinaryChoice { Zero, One }

match num {
    BinaryChoice::Zero => ...,
    BinaryChoice::One => ...
} // Exhaustive!
``` 

Wait, how do I encode randomness? 

**Random value -> Type-safe wrapper**

```rust
// Define the type that encodes the constraint
enum BinaryChoice {
    Zero,
    One,
}

impl BinaryChoice {
    // Generate random instance 
    fn random() -> Self {
        match rand::thread_rng().gen_range(0..2) {
            0 => BinaryChoice::Zero,
            1 => BinaryChoice::One,
            _ => unreachable!(),
        }
    }
}

// Usage - now type-safe!
let num = BinaryChoice::random();
match num {
    BinaryChoice::Zero => Action::Reveal,
    BinaryChoice::One => Action::Flag,
}  // Exhaustive!
```

Feels like an overkill...but let's come back to this option later.

2. Unreachable with wildcard

```rust
match get_range(0..2) {
    0 => ...,
    1 => ...,
    _ => unreachable!("I know this is - or 1.")
}
```

3. Runtime assertion

```rust
let num = get_range(0..2);
assert!(num == 0 || num == 1); // runtime check

match num {
    0 => ...,
    1 => ...,
    _ => unreachable!(),
}
```

**Among the three options, the most idiomatic and safest may be Option 1: Using a type!**  

Why may it be most idiomatic? 
- "Make invalid states unrepresentable" &mdash; core Rust philosophy 
- Leverages the type system (Rust's strength)
- Self-documenting `Action::random()` is clear
- Compiler-verified exhaustiveness
- Follows Rust patterns like `Option::unwrap()`, `Result::ok()`, etc.
- Reusable 

Why is it the safest? 
- **Compile-time safety**: compiler enforces exhaustiveness
- **No panic possible**: no `unreachable!()` to accidentally hit
- **Refactor-safe**: if you change to `num = get_range(0..3)`, compiler forces me to handle it everywhere
- **future-proof**: if `gen_range` behavior changes, I handle it in one place
- **No assertion overhead**: assertions have runtime cost (even if small)



