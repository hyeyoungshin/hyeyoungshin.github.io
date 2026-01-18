---
layout: post
title: "to match or not to match"
date: 2026-01-18
---

### To match or not to match

In science, questions tend to have definitive answers—unlike in literature or philosophy, where interpretation reigns. This is a broad generalization, but it's one of the things I've always appreciated about studying science: there's truth, and there's the whole truth.

So where is this coming from?

I was recently implementing a filter on `Vec<T>` for some type `T`, and realized there are multiple valid solutions. Should I consume the vector? Return references? Use iterators? Each approach is "correct" but optimizes for different things—readability, performance, memory usage. It's a reminder that even in the seemingly objective world of programming, there's rarely just one truth.

In pseudo code:

```rust
// some_vec: Vec<T>
// bag: HashMap<T, V>
some_vec.iter()
       .filter(|item| bag.get(item).unwrap() == SomeEnum)
``` 

This doesn't work. Let's analyze it by looking at types!  

`some_vec.iter()` returns something of type `Iter<_, T>`.  

The `'_` in `Iter<'_, T>` is a **lifetime placeholder** that means "there's a lifetime here, but Rust can infer it, so it doesn't have to be explicit." The `'_` specifically says "This iterator borrows from something with some lifetime, but let the compiler figure out what that lifetime is." as opposed to explicit lifetime like in `Iter<'a, T>`.  

**In practice:**
You rarely need to think about `'_` &ndash; it's just Rust's way of saying "trust me, I know there's a lifetime here and I've checked it's valid." It's syntactic sugar that makes code cleaner than writing explicit lifetime parameters everywhere.  

Next, let's look at `item`'s type inside the closure. The compiler says it has type `&&T`. Why?  
Because the closure is capturing `item` **by reference**.  

When I write `|item|`, Rust infers whether to
- Move the value into the closure
- Borrow it by reference  

For iterators, the closure parameter in `filter` is often captured by reference, so
- The iterator yields `&Coordinate`
- The closure parameter `|item` captures it as `&(&T)` ergo `&&T`. 

Here comes the point

There are different approaches to write the code with the same semantics:

```rust
some_vec.iter().filter(|item| bag.get(item).unwrap() == &SomeEnum)    // 1. Let closure capture by reference
some_vec.iter().filter(|&item| bag.get(item).unwrap() == &SomeEnum)   // 2. Pattern match once 
some_vec.iter().filter(|&&item| bag.get(item).unwrap() == &SomeEnum)  // 3. Pattern match twice
some_vec.iter().filter(|item| bag.get(*item).unwrap() == &SomeEnum)   // 4. Explicit deref in body
some_vec.iter().filter(|item| *bag.get(item).unwrap() == SomeEnum)    // 5. Explicit deref in body version 2
```

All of these work because how flexible `get` is. 

```rust
       pub fn get<Q>(&self, k: &Q) -> Option<&V>
       where
       K: Borrow<Q>,
       Q: Hash + Eq + ?Sized,
 ```

  `get` takes a reference type `&Q`. And thanks to the `Borrow` trait, _it can also accept references to references_. Rust automatically dereferences `&&Q` to `&Q`.  

```rust
       // These all work:
       bag.get(item)           // item: &&T
       bag.get(*item)          // item: &T
       bag.get(&**item)        // item: &T
```

I can also use macro:

```rust
some_vec.iter().filter(|item| matches!(bag.get(item).unwrap(), SomeEnum))
```

Using the macro is nice because:
- No dereferencing needed in the code itself
- More readable for complex patterns
- Works well with variants that have data. For example, `matches!(bag.get(item).unwrap(), SomeEnum(data) if data > 0)`


## Performance Analysis

In practice: 1,2,3, and 4 are identical performance after optimization.

1. **The dereferencing is "free"** 
  - Dereferencing a reference (`*item` or pattern matching `&item`) is just a pointer follow. The optimizer eliminates any overhead.

2. **`HashMap::get`** handles all reference types efficiently
  - The `Borrow` trait and compiler optimizations make these equivalent. The hash lookup is the expensive part, not the dereferencing. 

3. **Copying `item` vs borrowing it**

```rust
.filter(|&&item| ...)   // item is copied
```
- The only potential difference in performance is in 3 &mdash; dereferencing twice. 
- Dereferencing twice here means "give me the actual value, not a reference to it", so Rust must either  
  + **Copy** it (if `Copy` trait exists)   
  + **Move** it (not applicable here &mdash; can't move out of borrowed data)
- If `item` is small, no difference (copying 8 bytes is as fast as passing a pointer)
- If `item` is large, _slightly slower_ but still negligible in practice. Modern CPUs handle small struct copies incredibly efficiently.

_With optimizations enabled (`--release`), the compiler likely generates identical machine code for all approaches anyway_

So back to the earlier question, to match or not to match, **choose based on clarity, not performance.** 

Maybe I will choose this, which is idiomatic, clear, and handles `Option` safely.

```rust
.filter(|&item| matches!(bag.get(item), Some(&SomeEnum))) 
``` 

