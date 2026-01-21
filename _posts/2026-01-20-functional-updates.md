---
layout: post
title: "functional updates"
date: 2026-01-20
---

I'm a big fan of functional programming, so naturally I want to handle state updates functionally. Coming from high-level languages, like Scala, where immutability was abstracted away, I thought it was enough to write functions that return new data rather than mutating existing data. But working in Rust forced me to understand the underlying mechanics: the distinction between **consuming ownership** and returning new values.

### Consuming vs Borrowing when Updating
- **Take ownership (or consume) with `self`**  
Take ownership, bind as mutable. 

- **Borrow: I'm forced to clone!**  
Since I only have `&self` (a reference), I can't move the data out. To create a new `SomeData`, I must clone it!

```rust
impl SomeData {
    // Takes ownership - consumes the board!
    pub fn update_1(self, x: T) -> SomeData {
        let mut data = self.data; // Just move, no clone!
        data.insert(x);
        
        SomeData { data } 
    }
    
    // Borrows - board can still be used after
    pub fn update_2(&self, x: T) -> SomeData {
        let mut new_data = self.clone(); // Must clone!
        new_data.data.insert(x);
        
        new_data
    }
}
```
**What happens with `self`:**
```rust
let some_data = SomeData::new(...);

let new_data = some_data.update_1(x);  // some_data moved
let newer_data = new_data.update_1(y); // some_data is NOT available!
```
This is the **builder pattern** or **functional update pattern** &mdash; returning a new/modified board each time.

**What happens with `&self`:**

```rust
let new_data = some_data.update_2(x);    // some_data cloned
let newer_data = some_data.update_2(y);  // some_data is still available!
```

### Mutable/imperative style (modify in place)

**What happens with `&mut self`:**

```rust
impl SomeData {
    pub fn update(&mut self, x: T) {
        self.data.insert(x)   // modify self directly
    }
}

// Usage:
let mut some_data = SomeData::new(...);
some_data.update(x);
some_data.update(x);
```

### Rule of Thumb

- use `self` for **transforming** and **updating**
- use `&self` for **reading data** (query/getter)
- use `&mut self` to modify in place, if that's what you are into