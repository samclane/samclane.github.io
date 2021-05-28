---
layout: post
comments: true
published: true
title: Surreal Numbers In Python
description: 'Part 1: Intro'
---
## Motivation

My goal in this post is to show how an abstract mathmatical conccept, such as [Surreal Numbers](https://en.wikipedia.org/wiki/Surreal_number), can be captured and explored in Python. This is not purely instructional; rather it's an open-book experiment with Python that can hopefully bridge the gap in our knowledge.

## What are surreal numbers?

![]({{site.baseurl}}/https://upload.wikimedia.org/wikipedia/commons/thumb/4/49/Surreal_number_tree.svg/800px-Surreal_number_tree.svg.png)

Surreal numbers are, in a criminally brief sense, a class of numbers that subsumes the Real Numbers via the inclusion of the infinite/infantesimals, while retaining a lot of properties of the Real Numbers, such as arithmetic and ordering (`<`, `=`, `>`, etc.). However, I'm less interested in their mathematical properties, as their construction and testing. Encapsulating their production rules and evaluating some of their inductive properties can be extremely useful, as if you can generate the surreals, generating any other "number" is trivial. Surreals also have applications in [Game Theory](https://en.wikipedia.org/wiki/Combinatorial_game_theory) and [Nonstandard Analysis](https://en.wikipedia.org/wiki/Nonstandard_analysis), which are of interest to other Computer Science disicplines. 

## Getting Started


We will be using [this whitepaper](https://www.whitman.edu/documents/Academics/Mathematics/Grimm.pdf) from Gretchen Grimm, as it does a good job of succintly going over the concepts in Knuth's book.

To start, we'll use this small example from [this YouTube video](https://www.youtube.com/watch?v=OWnm79mEiCY). To generate Surreal Numbers, we'll start with two of Conway's Axioms:

```
1. Every number corresponds to two sets of previously created numbers, such that no
member of the left set is greater than or equal to any member of the right set.
```

```
2. One number is less than or equal to another number if and only if no member of the
first number’s left set is greater than or equal to the second number, and no member of the second
number’s right set is less than or equal to the first number.
```

We'll generate a set of numbers from the previous set. The iterations are referred to as "days", with the day as number is generated referred to as its "birthday". Getting started is a little bit tricky, but luckily Python provides some data structures that are isomorphic to our needs. Since we're computer scientists, we'll start on the **zeroth day**. We don't have any previous numbers, so our only option is the *empty set*, `()`, which is easily represented in Python by an empty `tuple`. This represents the number `0`, or as a surreal form, `{|}`. With our first actual number, we can apply our 2 rules to get `{0|}` and `{|0}`, corresponding to 1 and -1 respectively. 

<iframe width="784" height="144" src="https://datalore.jetbrains.com/view/embed/y0irTQxpwjtJraOPVB5Kuf/0?height=144" frameborder="0"></iframe>

We'll also need code to formalize the second axiom- the defintiion of less-than-or-equal-to. From this rule we can begin to "order" our numbers.

<iframe width="784" height="112" src="https://datalore.jetbrains.com/view/embed/y0irTQxpwjtJraOPVB5Kuf/1?height=112" frameborder="0"></iframe>

We can institute some tests in order to prove (especially to ourselves) that this implementation works, and is consistent with the Axioms presented.

<iframe width="784" height="176" src="https://datalore.jetbrains.com/view/embed/y0irTQxpwjtJraOPVB5Kuf/2?height=176" frameborder="0"></iframe>

However, as we begin to grow our set of Surreals, which, by the 2nd day, will have grown from 3 to 20, we'll need a more intelligent way of generating and storing them. You may be thinking of "dunder" or "magic" class methods, and we'll begin exploring encapsulating Surreals in the next post.

---

A link to the full Datalore Notebook can be found [here](https://datalore.jetbrains.com/view/notebook/y0irTQxpwjtJraOPVB5Kuf).
