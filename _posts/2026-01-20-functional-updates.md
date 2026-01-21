---
layout: post
title: "Functional Updates"
date: 2026-01-20
---

## Functional Updates
I'm a big fan of functional programming, so naturally I want to handle state updates functionally. Coming from high-level languages, like Scala, where immutability was abstracted away, I thought it was enough to write functions that return new data rather than mutating existing data. But working in Rust forced me to understand the underlying mechanics: the distinction between **consuming ownership** and returning new values.

### Consuming vs Borrowing

```rust
impl SomeData {
    // Takes ownership - consumes the board!
    pub fn is_something(self, x: &T) -> bool {
        self.data.contains(x)
    }
    
    // Borrows - board can still be used after
    pub fn is_something(&self, x: &T) -> bool {
        self.data.contains(x)
    }
}
```

The implication of passing `self` instead of `&self` explains the difference between "consuming" and "borrowing".

**What happens with `self`:**

```rust
let some_data = SomeData::new(...);

some_data.is_something(&x);  // some_data is MOVED here and destroyed!
some_data.is_something(&x);  // ***Error: some_data was already moved
```

**What happens with `&self`:**

```rust
let some_data = SomeData::new(...);

some_data.is_something(&x);  // Borrows some_data temporarily
some_data.is_something(&x);  // Works! (can use some_data again)
```

This is the **builder pattern** or **functional update pattern** &mdash; returning a new/modified board each time.

### Mutable/imperative style (modify in place)

**What happens with `&mut self`:**

```rust
impl SomeData {
    pub fn update(&mut self, x: &T) {
        self.data.insert(x)   // modify self directly
    }
}

// Usage:
let mut some_data = SomeData::new(...);
some_data.update(&x);
some_data.update(&x);
```

### Rule of Thumb

- use `self` for functional updates (return new data each time)
- use `&self` for methods that just read data
- use `&mut self` to modify in place if that's what you are into