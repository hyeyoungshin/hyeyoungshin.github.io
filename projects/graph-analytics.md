---
layout: default
title: Graph Analytics Library
---

<!-- [← Back](/) -->

# Graph Analytics Library

**Logic Programming · Graph Theory · Data Analytics · Performance Optimization**

## What problem RelationalAI is solving
Traditional databases store data in silos. Even after companies consolidate their data into a single place (like a data lake), the knowledge, the business logic and relationships between data, still ends up scattered across hundreds of custom applications. RelationalAI's goal is to bring that knowledge into the database itself, so that reasoning over interconnected data can happen in one unified system.

## The three pillars

1. **The Knowledge Graph**:
A relational knowledge graph represents data as a collection of relations, nodes, edges, and hyperedges, stored in a fully normalized form. Think of it as a database where the relationships between data points are first-class citizens, not an afterthought. Instead of joining tables in SQL, you model your domain as a graph of interconnected facts and query over those connections directly.

2. **The Engine**: aka the Relational Knowledge Graph System (RKGS) combines the relational paradigm with knowledge graphs and uses a query optimizer to automate complex tasks like algorithm selection. One of its technical innovations is a new join algorithm (called "dovetail") that makes querying fully normalized graph data efficient at scale, something traditional databases struggle with.

3. **Rel, the language**: Rel is a declarative modeling language built around four main constructs, identifier, tupling, abstraction, and application, drawing on decades of research in both databases and programming languages. Unlike SQL, which is a sublanguage (designed only for querying), Rel is designed to express the entire application logic,  queries, rules, derived knowledge, and reasoningm all in one place. It's rooted in first-order logic and predicate calculus, which is why it appealed to people with a PL background.


## How my work fits into this
I was part of a small team building the graph analytics library. The library sits on top of this stack, implemented in Rel and exposed via a Python API. The engine team was responsible for the core query execution and optimization infrastructure. The language team owned Rel itself.  
My role sat at the intersection: I built graph algorithms (like shortest path, PageRank, centrality) that ran within the RKGS, benchmarked them against industry competitors, and worked with both teams when bugs or performance issues crossed boundaries.
