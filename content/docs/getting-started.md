---
title: "Getting Started"
date: 2026-04-13
weight: 1
---

## Install

```bash
# coming soon
meshdb --version
```

## Your first graph

```cypher
CREATE (a:Person {name: "Ada"})-[:KNOWS]->(b:Person {name: "Grace"})
RETURN a, b;
```
