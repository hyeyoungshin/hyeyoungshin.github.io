---
layout: post
title: "expression-statement"
date: 2026-02-04
---

Statements and expressions are two fundamental concepts in programming. Coming from other languages, I understood them as:
- Statements do not return a value and typically end with `;`
- Expressions evaluate to a value and do not end with `;`

However, I found this explanation incomplete when reading the Rust Programming Language book, particularly regarding this sentence:

> Calling a macro is an expression 

I've been using the `println!` macro thinking when I use it, it's a statement. As also noted by the `;` at the end of it.

```rust
println!("hello");
``` 

Running this code also does not return anything. So I was a little confused.  

It turns out that the above example is called an "expression-statement". 

### Expression Statement 

"Expression-statement" is a term from programming language theory and compiler design. They are expressions that are turned into statements by adding `;` at the end. 

`println!("hello")` without the semicolon is an **expression** that returns a value &mdash; specifically, it returns the unit type `()`. The semicolon at the end is actually **turning that expression into a statement**.

```rust
// This is an expression that evaluates to ()
println!("hello")

// This is an expression-statement (the ; converts it)
println!("hello");
```

In Rust, every expression evaluates to some value. When there's no meaningful value to return, Rust uses the unit type `()`, which represents "nothing" or "no meaningful value." 
This is different from truly returning nothing &mdash; it's returning a specific type that says "_I completed but have no data to give you._"

Consider the following example:

```rust
let x = println!("hello");
// x has type (), the unit type
```

This compiles fine because `println!` does return something &mdash; just not something useful.

The semicolon at the end is what makes `println!("hello");` a statement rather than leaving it as an expression. Without the semicolon in certain contexts like at the end of a function, I'd be returning `()` as the function's return value.

So, the book is correct (obviously): calling a macro _is_ an expression. It's just that many macros like `println!` evaluates to `()`. The statement part comes from adding the semicolon, which discards that `()` value and turns the whole thing into an **expression-statement**.

## What are _true_ statements in Rust? 

1. `let` bindings (variable declarations)

```rust
let x = 5; 
let mut y = 10; 
let z: i32 = 33;
```

2. **Item declarations** (functions, structs, enums, etc)

```rust
fn my_function() {
    //,,,
}

struct MyStruct {
    field: i32,
}

enum MyEnum {
    Variant1,
    Variant2,
}

impl MyStruct {
    //...
}
```

These are the only true statements in Rust that don't evaluate to any value &mdash; not even `()`. They define things rather than compute values.

Everything else I might think of as a "statement" is actually an expression or an expression-statement (an expression followed by a semicolon).

**Summary**

- **True statements**: `let` bindings and item declarations (no value at all)
- **Expressions**: everything else (always evaluates to some value)
- **Expression-statements**: expressions with a semicolon that discards their value

This is why Rust is often called an "expression-oriented language" &mdash; almost everything produces a value!  


Ok, so I get that. Then how do you explain this?

```rust

let result = loop {
    counter += 1;

    if counter == 10 {
        break counter * 2; // break "carries" the value counter * 2
    }
};

```
`result` is bound to the value 20 after the loop which is the result of evaluating `counter * 2`. Then it looks like `break counter * 2;` is returning a value. 

Well, `break counter * 2`; is another expression-statement.

- `counter * 2` is an expression
- `break counter * 2` is a `break` expression that "carries the value `counter * 2` out of the loop
- The semicolon makes it an expression-statement

If you look at the type of the `break` expression itself, it's actually `!` (the "never" type), because `break` never returns to the point where it was called &mdash; it jumps out of the loop. But it does _carry a value_ that becomes the value of the loop expression.

The important thing to understand is that the semicolon doesn't "discard" the break value the way it would discard the value of a normal expression.

The `!` (never) type doesn't mean "doesn't return anything" &mdash; it means "never returns at all" in the sense that execution never continues past this point. It's about control flow, not about values. This is special behavior for control flow expressions like `break` and `return`. The semicolon is conventional/stylistic, but the value is still carried through or even returned (to the loop).
