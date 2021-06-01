---
layout: post
comments: true
published: false
title: Surreal Numbers in Python 2
description: Object Oriented Approach and First Roadblocks
---
## Recap

In the last post, we started creating the Surreal Numbers with a Functional approach, going along with the [Grimm Whitepaper](https://www.whitman.edu/Documents/Academics/Mathematics/Grimm.pdf). However, even with only 3 Numbers created, we have lots of data to keep straight. Perhaps some encapsulation would be handy...

### Making a class

We'll start with a simple class declaration: every Surreal has a "day" it was born on, and consists of a Left and Right set of numbers.

```python
class Surreal:
    birthday=None
    left_set=()
    right_set=()
```
