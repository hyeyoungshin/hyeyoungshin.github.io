---
layout: default
title: Secure Compilation Verification
---

# Secure Compilation Verification

**Formal Verification · Compilation Theory · Security Analysis**

## The problem 

When you write code in a high-level language, you reason about your program using the guarantees that language provides. For example, Coq is a total language — every program is guaranteed to terminate. So two functions that behave identically on all valid inputs are considered equivalent, and a Coq programmer can safely reason about their program based on that equivalence.

The problem is that Coq compiles down to OCaml, which is a partial language and programs can loop forever or diverge. When your carefully reasoned source-level equivalences get compiled into OCaml, a malicious or unexpected OCaml context can break those equivalences by passing in a non-terminating argument. This can cause subtle bugs and, more seriously, security vulnerabilities, because the guarantees you relied on in the source language no longer hold in the compiled target.

## What fully abstract compilation means

A compiler is fully abstract if it preserves and reflects equivalences between programs, meaning two source programs are equivalent if and only if their compiled versions are equivalent. This is a strong correctness property: it means the compiler doesn't leak information or introduce behaviors that weren't observable in the source.

Prior work had shown fully abstract compilation between languages that are either both total or both partial. My work tackled the harder and more practically relevant case: compiling from a total source to a partial target exactly what happens in Coq-to-OCaml compilation.

## The approach
The key insight is in the target language design. Rather than compiling into plain OCaml-like STLC, you compile into a richer target language that adds two things:

- **Recursive types** to model potentially non-terminating computation
- **Modal types with effect annotations** types that explicitly track whether a computation is pure (guaranteed to terminate, marked ◦) or impure (possibly non-terminating, marked •)

The compiler itself is remarkably simple. It's essentially the identity on terms, just annotating types with purity information. The key payoff is that the modal type system prevents compiled source programs from ever being linked with impure, non-terminating target contexts. The adversarial argument that would break the equivalences simply has the wrong type and can't be linked in.

## The proof technique
The standard way to prove full abstraction uses a technique called back-translation; you take a target context that breaks an equivalence and translate it back into a source context. But this is very hard when your target has non-termination and your source doesn't, because you can't back-translate a diverging computation into a total language.

My novel contribution was avoiding back-translation entirely. Instead, I used the fact that in the simple source language (STLC with unit and booleans), contextual equivalence coincides with βη-equality, a much simpler, syntactic notion of equivalence that's easier to work with. On the target side, I defined a logical relation that is sound and complete with respect to contextual equivalence. This let me prove equivalence preservation by showing that βη-equality in the source implies logical relatedness in the target, sidestepping the back-translation problem altogether.

## What came out of it
I submitted an [abtract](https://github.com/hyeyoungshin/popl19src/blob/master/popl19src.pdf) to POPL student competition 2019 and received insightful reviews. 