---
layout: post
comments: true
published: true
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

We can add these shortcuts to our `Surreal` class as follows

```python
    @property
    def xl(self):
        return max(self.left_set or (None,))

    @property
    def xr(self):
        return min(self.right_set or (None,))
```

## Rationals

Great, so we can generate all the integers. But even grade-schoolers know about decimals/fractions. Where do those numbers come from in our system? If we remember the bathroom-scale analog, we can think of the `|` (pipe) as a sort of arrow/indicator, where we can see surrounding numbers to generate further ones. We can start to make some rational numbers by looking "where they live", in-between integers.

Thus, we can think of the following surreal, `{0|1}`, as "the number between 0 and 1", or `1/2`. Similarly, `-1/2` is represented by `{-1|0}`. We can now add these rationals to our forms, to look even "further" in. 

`{1/2|1} = 3/4`, born on day 3. `{1/2|3/4} = 5/8`. If you notice, our denominator will always be a power-of-2, so we call these the `Dyadic Rationals`. 

## Actually coding this

Conway uses a recursive function to define his conversion. It follows the common logic of splitting the problem into the base-cases and recursive-case. With surreals, the base cases are `-1, 0, and 1`. As such, we'll include special `classmethods` to act as constructors for these special cases.

```python
    @classmethod
    def zero(cls):
        return cls(0, nothing, nothing)

    @classmethod
    def one(cls):
        return cls(1, (0,), nothing)

    @classmethod
    def neg_one(cls):
        return cls(1, nothing, (0,))
```

We can then start to write a `__float__` function, allowing for easy conversion of Surreals to their Real Number counterparts. We'll start by coding the base-cases:

```python
    def __float__(self):
        if self == Surreal.zero():
            return 0.
        elif self == Surreal.one():
            return 1.
        elif self == Surreal.neg_one():
            return -1.
        else:
        	raise NotImplementedError()
```

In order to cover the rest of the number-line, we need to come up with a general-purpose algorithm. Remember  previously, we defined 3 cases where we could make a "form" for our collection: left-number, right-number, and in-between-number, where the first 2 increment/decrement the number, and the last finds the number in-between the two numbers given.

```python
		...
        else:
            if len(self.left_set) == 0:  # Left number
                return min(self.right_set) - 1
            elif len(self.right_set) == 0:  # Right number
                return max(self.left_set) + 1
            else:  # In-between
                return (min(self.right_set) + max(self.left_set)) / 2
```

## Going back

We should be able to do the inverse of float-conversion, where we can pass a number to our class, and it converts it to a valid Surreal. However, we can't simply go from a decimal number to a Surreal; we're only able to generate numbers from a certain domain, so it stands to reason that using the same rules, we can only convert certain numbers back into Surreals. We can generate all integers (`k(n+1) = k(n) + 1`) and all dyadic rationals (`k(n+1) = (2(k(n) + 1) / 2^n`).

To start, we'll use a simpler example that follows an easy pattern- the integers. We know that a positive integer-number `i` can be expressed in Surreal form as `{i-1|}`. Negative integers simply flip which set is used, as `i === {|i+1}`. Encapsulating our base-case of `zero`, we get the following code:

```python

    @classmethod
    def from_int(cls, i: int):
        if i == 0:
            return Surreal.zero()
        elif i > 0:
            return Surreal(abs(i), (i-1,), nothing)
        elif i < 0:
            return Surreal(abs(i), nothing,(i+1,))
        else:
            raise Exception("NaN")
```

<iframe width="784" height="145" src="https://datalore.jetbrains.com/view/embed/y0irTQxpwjtJraOPVB5Kuf/22?height=145" frameborder="0"></iframe>

Dyadics are a bit harder than the plain integers, as one needs to find an odd-integer numerator N, and a denominator that's a power-of-2. Without going through the details of solving the equations, we find that the surreal form of dyadic `x` is ` { x - 1/2^{k} | x + 1/2^{k} }`. This will produce a number whose numeric-average is equal to `x`. 

` (x - 1/2^{k}) + (x + 1/2^{k})/2 = 2x/2 = x`

We can use Python's built-in `Fraction` class to help conver to rational-fractions, and store the `numerator` and `denominator` separately. We can also use `math.log2` to help us find `k`. 


```python

    @classmethod
    def from_dyadic(cls, f: Union[Fraction, float]):
        if isinstance(f, float):
            f = Fraction(Decimal(f))
        k = log2(f.denominator)
        n = abs(f.numerator//2)+1
        return Surreal(n, (f - (1/2)**k,), (f + (1/2)**k,))
```

To check, we can `assert` some quick tests:

<iframe width="784" height="138" src="https://datalore.jetbrains.com/view/embed/y0irTQxpwjtJraOPVB5Kuf/24?height=138" frameborder="0"></iframe>

<iframe width="784" height="128" src="https://datalore.jetbrains.com/view/embed/y0irTQxpwjtJraOPVB5Kuf/29?height=128" frameborder="0"></iframe>

<iframe width="784" height="129" src="https://datalore.jetbrains.com/view/embed/y0irTQxpwjtJraOPVB5Kuf/30?height=129" frameborder="0"></iframe>

With all this in place, we can finally create a Surreal generation-function that includes the dyadic-rationals:

<iframe width="784" height="929" src="https://datalore.jetbrains.com/view/embed/y0irTQxpwjtJraOPVB5Kuf/34?height=929" frameborder="0"></iframe>

---

Full Jupyter notebook [here](https://datalore.jetbrains.com/view/notebook/y0irTQxpwjtJraOPVB5Kuf)