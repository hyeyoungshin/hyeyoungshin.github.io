---
layout: default
title: Data-Driven Testing Tool for R
---

[← Back](/)

# Data-Driven Testing Tool for R

**R · Testing Frameworks · Data Analysis · Statistical Computing**

## The problem our team was solving
R is a dynamically typed language, which means functions don't declare what types they accept or return. This makes it very hard to reason about correctness statically. The team wanted to understand the real-world behavior of R functions, specifically, what types of inputs they actually accept in order to inform the design of a type system for R. To do that, we needed to generate as many diverse, realistic function calls as possible.

## The core idea
Rather than generating random inputs from scratch (which doesn't work well for dynamic languages because the inputs need to be realistic and complex), we took a data-driven approach: first observe what values real R programs actually use, store them in a database, and then use those observed values as the raw material for fuzzing.

The pipeline has two phases

**Phase 1: Recording**

We ran large amounts of existing R executable code, examples, tests, and documentation scripts from R packages on CRAN, and traced all function calls, capturing every argument and return value. These values were stored in a custom database called `sxpdb`, which holds millions of unique R values along with metadata about their types, structure, and origin.

**Phase 2: Fuzzing**

For each function we wanted to test, the fuzzer drew inputs from the database — values that were previously observed in real code — and called the function with those inputs. If a call succeeded (no errors, warnings, or crashes), it was recorded as a new call signature: a description of what types the function accepted and returned. The fuzzer gradually narrowed its search over time, starting broad and becoming more targeted as it accumulated successful calls.

## Why it's technically interesting
A few things make this non-trivial:  

**The database design** is custom-built for R's value model. It handles R's copy-on-write semantics, lazy evaluation, and the fact that R values carry rich metadata like attributes and class names. It uses fast hashing (XXH128) and compressed bitsets for efficient lookup.

**The fuzzing strategy** is closer to mutation-based fuzzing than pure random fuzzing. It uses previously successful calls as seeds and relaxes search parameters gradually, which is more effective than random generation for a language like R.

**The tracer** hooks into an extended R virtual machine at the function exit and context jump events to capture calls without missing them even when R uses long jumps for control flow.


## What the results showed
Fuzzing 4,829 R functions generated over 1.2 million new unique call signatures, about 105 times more than tracing alone could discover. It also improved code coverage by around 20% for the functions it could fuzz, and uncovered a real memory corruption bug in a widely used R package.

## How my work fits into this
I contributed to the earlier stages of the project, the recording infrastructure, the data collection and visualization, and the tracing machinery before the project continued beyond my time there.

For the visualization work, I used R and ggplot while working through Hadley Wickham's R for Data Science, which gave me a solid grounding in how to approach exploratory data analysis and build clean, informative plots. We were working with tens of millions of recorded values, so the visualizations were important for understanding the shape and distribution of the data, things like what types of values were most common, how they were distributed across packages, and where the interesting or unusual cases were.