---
layout: post
title: "never type"
date: 2026-02-01
---

I learned about type `!` today when I puzzled over the following `match` block.

```rust
match parse_action(player_input) {
    Ok(action) => action,           // Type: Action
    Err(msg) => {
        println!("{}", msg);
        continue;                    // Type: ! (never)
    }
}
```

The match arms don't _seem to_ have the same type, but Rust accepts the code. Actually, they do have compatible types in a non-obvious way. 

### The Never Type `!`

`continue`, `break`, `return`, and `panic!` all have the special type `!` (pronounced "never"), which means "this code **never** returns normally".

The `!` type is compatible with **any other type** because it represents "this branch never completes".

```rust
let parsed_action: Action = match parse_action(player_input) {
    Ok(action) => action,      // Returns Action
    Err(msg) => {
        println!("{}", msg);
        continue;               // Never returns (jumps back to loop start)
    }
};
// parsed_action has type Action
```

Looking at the code again, since the `Err` branch **never completes** (it jumps to the next loop iteration), Rust knows that if we get past the match, we **must** have taken the `Ok` branch, so `parsed_action` is definitely an `Action`. 

Without `continue`, the code would fail. The `Err` branch would have type `()` (the unit type).

## The Pattern

This is a very common Rust pattern.

```rust
loop {
    let value = match try_something() {
        Ok(v) => v,              // Extract the value
        Err(e) => {
            handle_error(e);
            continue;            // Try again
        }
    };
    
    // Use value here (guaranteed to be Ok variant)
    process(value);
}
```


Another common pattern with `return`:

```rust
fn process_input(input: &str) -> Result<(), String> {
    let parsed = match parse(input) {
        Ok(val) => val,
        Err(e) => return Err(e),  // Early return with !
    };
    
    // parsed has the Ok type here
    do_something(parsed);
    Ok(())
}
```

Borrowing terminologies from type theory, Rust's Never Type is the subtype of every other type. In other words, a **bottom** type. This allows _things_ of type `!` to be coerced into any other type. 

It's also an **empty** type or **uninhabited** type &mdash; no _value_ has type `!`. Wait, I just said _things_ of type `!` is coerced into any other type in a couple of sentences ago. That's right. I chose an ambiguous term "things" on purpose! 

These things are not anything of **value**. They are expressions that would **never complete**, meaning they do not produce a value. These expressions represents
- divergence
- useful coercion in branching
- unreachable code path
