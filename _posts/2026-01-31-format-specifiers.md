---
layout: post
title: "about format specifiers"
date: 2026-01-31
---

I've been printing values with `println!` placeholders like `{}` without really understanding what they work. Today I decided to look further into it. 

The placeholder `{}` works through the `Display` **trait**. It can print the value of a variable inside it. 

Examples from the book:

```rust
let x = 5;
println!("x = {x}");
```
This code will print 

```
x = 5
```

It can also print the result of evaluating an expression.

```rust
let x = 5;
let y = 10;

println!("x = {x} and y + 2 = {}", y + 2);
```
Placing empty curly brackets in the format string followed by a comma-separated list of expressions will print the result of evaluating each expression in the same order. 

This will print
```
x = 5 and y + 2 = 12
```

Well, this seems simple enough. How about printing something a value of more complex types? 

Let's say I have a datatype `Player`:

### Option1: Use `{:?}` for Debug formatting

```rust
#[derive(Debug)]
struct Player {
    id: u32,
    name: String,
}

let player = Player { id: 1, name: "hyeyoung".to_string() };
println!("{:?}", player);
``` 
This will print 
```
Player { id: 1, name: "alice" }.
```

As shown above, using `{:?}` format specifier works with any type that implements `Debug`.

### Option 2: Use `{:#?}` for pretty-print Debug

```rust
let players = vec![
    Player { id: 1, name: "alice".to_string() },
    Player { id: 2, name: "bob".to_string() },
];

println!("{:#?}", players);
```

This will print
```
[
    Player {
        id: 1,
        name: "alice",
    },
    Player {
        id: 2,
        name: "bob",
    },
]
```

### Option 3: Implement `Display` for custom formatting

```rust
use std::fmt;

impl fmt::Display for Player {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Player {} (ID: {})", self.name, self.id)
    }
}

let player = Player { id: 1, name: "alice".to_string() };
println!("{}", player);  
```

This will print
```
Player alice (ID: 1)
```

**Format Specifiers**
| Specifier | Trait | Use case | Example output |
|---|---|---|---|
| `{}` | `Display` | User-facing output | `Player alice (ID: 1)`|
| `{:?}` | `Debug` | Developer debugging | `Player { id: 1, name: "alice" }` |
| `{:#?}` | `Debug` | Pretty debugging | Multi-line formatted |
| `{:p}` | `Pointer` | Memory address | `0x7ffd5e123456` |
| `{:b}` | `Binary` | Binary format | `101000` |
| `{:x}` | `LowerHex` | Hex format | `2a` |

Most of the types I use already have `Debug` derived. If I want nicer output, I can implement `Display`.

```rust
use std::fmt;

impl fmt::Display for Player {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{} ({}pts)", self.name, self.points)
    }
}

// Now I can just use {}
println!("Player: {}", player);     // prints "Player: alice (0pts)"
```
Using both `Debug` and `Display` is a common pattern. `{:?}` for debugging and `{}` for displaying for users. 

**Summary:**
- `{}` requires `Display` for user-facing output
- `{:?}` requires `Debug` for developer debugging (derive this on all your types)
- `{:#?}` for pretty-printed `Debug`
- **Most of the time, just use {:?} for complex types**
- Only implement `Display` when you need custom user-facing output
