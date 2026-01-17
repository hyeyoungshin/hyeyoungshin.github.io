---
layout: post
title: "function overloading vs dynamic dispatch"
date: 2026-01-16
---

## Two concepts that I mixed up

I was writing a constructor for a datatype for one of my Rust projects. Then I thought of a scenario where having two functions that have the same name but with different parameters might be advantageous. This is not ambiguous because the compiler can know which one is being called based on the argument types the caller provides. I was trying to remember what this behavior is called... then it came to me "it's **Dynamic Dispatch**"! 

Of course, I was wrong. It has "Dynamic" in the name (eye rolling). So I did some reading and here is what I found:

### Function Overloading
**Same function name, different parameters** &ndash; the compiler decides which one to call at compile time based on the argument types. This is what I was thinking of. And _Rust DOES NOT support it!_

```rust
fn some_function(x: i32) { }
fn some_function(x: i32, y: i32) { }  // Same name, different params
```

### Dynamic Dispatch
**Choosing which method implementation to call at runtime** based on the actual type of an object, typically using traits.

```rust
trait Animal {
    fn speak(&self);
}

struct Dog;
impl Animal for Dog {
    fn speak(&self) { println!("Woof!"); }
}

struct Cat;
impl Animal for Cat {
    fn speak(&self) { println!("Meow!"); }
}

// Dynamic dispatch
fn make_sound(animal: &dyn Animal) {
    animal.speak();  // Don't know if it's Dog or Cat until runtime
}
```

**Key differences**  

| Feature | Function Overloading | Dynamic Dispatch |
|---|---|---|
| When decided | Compile time | Runtime |
| Based on | Parameter types/count | Actual object type |
| Rust support |  No | Yes (via traits)  |
| Performance | Fast (static) | Slightly slower (virtual call) |  


The following is something I should know:

### Static vs Dynamic Dispatch in Rust
```rust 
// Static dispatch (monomorphization) - fast
fn some_function<T: Animal>(animal: &T) {
    animal.speak();
}

// Dynamic dispatch - flexible
fn some_function(animal: &dyn Animal) {
    animal.speak();  // Runtime lookup
}
``` 

The first `some_function` is an example of static dispatch. Which `animal` to call `speak` on is known at compile time. The second `some_function` is dynamic dispatch as indicated also by the parameter type `&dyn Animal`.  

Looking at these examples leads me to wonder whether libraries typically provide both static and dynamic dispatch versions of functions like in the above example?

**The answer: Almost always static dispatch only**  
Why? Because static dispatch is more flexible (and faster!)
```rust
fn some_function<T: Animal>(animal: &T) {
    animal.speak();
}

// Caller can use it with concrete types (static dispatch)
let dog = Dog;
some_function(&dog);

// Caller can also use it with trait objects (dynamic dispatch)
let animal: &dyn Animal = &dog;
some_function(animal); // This works because &dyn Animal implements Animal
```
So one generic function covers both use cases.

**When do I need both:**
_Rarely_, when there's a performance-critical path or API design reason:
```rust
// rayon library example
pub fn par_iter<T>(items: &[T]) -> ParIter<T> { }  // Static
pub fn par_iter_dyn(items: &[&dyn Trait]) -> ParIterDyn { }  // Dynamic
``` 

**The guideline:**

- **Default**: Write generic functions with trait bounds `<T: Trait>`. They work for both cases.
- **Rarely**: Provide explicit `dyn Trait` versions only if there is specific need for special optimizations, different return types, API clarity. 




