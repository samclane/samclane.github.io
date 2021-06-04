---
layout: post
comments: true
published: false
title: Surreal Numbers In Python 3
description: Ratios and Rationalizations
---
## Generating Day 2

So far, we've generated 3 **forms** that we're familiar with: `0`, `1`, and `-1`, represented by `{|}`, `{0|}` and `{|0}` respectively. We also have a batch of new numbers, that we're not quite sure of:

```
{1|}{−1|} {0, 1|} {0, −1|} {−1, 1|}
{−1, 0, 1|} {|1} {| − 1} {|0, 1}
{|0, −1} {| − 1, 1} {| − 1, 0, 1} {−1|0}
{−1|1} {−1|0, 1} {0|1} {−1, 0|1}
```

Moving beyond Day 1, the numbers begin to follow a more intuitive pattern. We can read Surreal numbers as follows:

`If S is a surreal number consisting of {a|b}, then S is the number *in-between* a and b`. I'm going to start using a physical "number line" as an analog for the Real Numbers, as it's easier to concieve of the `|` in the surreal-notation to be akin to a marker on an analog scale,  where the number can be read by looking at the numbers surrounding the marker-line. Therefore, I will be using "left" and "right" as analogs for "less-than" and "greater-than" respectively. 

We can then conceptualize previous numbers, such as `{0|} is the number to the right of 0, which is 1`. We can then carry this rule further on. `If S = {1|}, then S is the number to the right of 1, which is 2`. We've now added `2` to our list of forms. Furthermore, we can take the negative of the statement. `If S = {|-1}, then S is the number to the left of -1, which is negative 2`. Another form for our collection, -2. 

By this point, you may have noticed that using entire sets for the left and right values of a surreal is a bit redundant. Equivalent forms yield equivalent numbers, so something like: 

`{-1, 0, 1|} === {1|} = 2` and `{|0, 1} === {|0} = -1` and finally `{-1, 0| 2} === {0|2} = 1` Essentially, we only care about `max(left)` and `min(right)` to define a number.

## Rationals

Great, so we can generate all the integers. But even grade-schoolers know about decimals/fractions. Where do those numbers come from in our system? If we remember the bathroom-scale analog, we can think of the `|` (pipe) as a sort of arrow/indicator, where we can see surrounding numbers to generate further ones. We can start to make some rational numbers by looking "where they live", in-between integers.

Thus, we can think of the following surreal, `{0|1}`, as "the number between 0 and 1", or `1/2`. Similarly, `-1/2` is represented by `{-1|0}`. We can now add these rationals to our forms, to look even "further" in. 

`{1/2|1} = 3/4`, born on day 3. `{1/2|3/4} = 5/8`. If you notice, our denominator will always be a power-of-2, so we call these the `Dyadic Rationals`. 