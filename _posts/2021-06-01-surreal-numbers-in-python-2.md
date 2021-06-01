---
layout: post
comments: true
published: true
title: Surreal Numbers in Python 2
description: Object Oriented Approach and Further Generation
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

When a Number is "created", we will know these values, so we'll include them in the constructor. We also want to sanity check `Axiom 1`, so we'll check that all Left Set values are less-than all Right Set values.

```python
    def __init__(self, birthday, left_set: tuple, right_set: tuple):
        for l in left_set:
            for r in right_set:
                assert l < r, "Not a Surreal Number: Violates Axiom 1"
        
        self.birthday = birthday
        assert self.birthday is not None, "Invalid Birthday Provided"

        self.left_set = left_set
        self.right_set = right_set
```

Let's include the definition of `less-than`, so we can begin to *order* our Surreal numbers. Recall Axiom 2:

> One number is less than or equal to another number if and only if no member of the
first number’s left set is greater than or equal to the second number, and no member of the second
number’s right set is less than or equal to the first number

We can add `less-than` as a dunder/magic method, so we can directly compare `Surreal` objects as Numbers

```python
    def __le__(self, other):

        for l in self.left_set:
            if l >= other: return False
        for ro in other.right_set:
            if self >= ro: return False

        return True
```

For convenience, let's also include a pretty print function in the form of a `__str__` method, so we can use the same symbolic convention as the paper:

```python
    def __str__(self):
        return "{" + ",".join(map(str, self.left_set)) + "|" + ",".join(map(str, self.right_set)) + "}"
```

Now that we have our basic functionality, we can try it out some of the previous Numbers we've created:

<iframe width="784" height="161" src="https://datalore.jetbrains.com/view/embed/y0irTQxpwjtJraOPVB5Kuf/7?height=161" frameborder="0"></iframe>

Let's ensure that the symbol for `0` looks right:

<iframe width="784" height="129" src="https://datalore.jetbrains.com/view/embed/y0irTQxpwjtJraOPVB5Kuf/8?height=129" frameborder="0"></iframe>

Then, let's manually create the other 2 numbers:

<iframe width="784" height="177" src="https://datalore.jetbrains.com/view/embed/y0irTQxpwjtJraOPVB5Kuf/9?height=177" frameborder="0"></iframe>

Another method of comparison for Surreals is known as "simplicity". One surreal is "simpler" than another if its birthday is less-than the others. For our example, `0` is the simplest, followed by `-1` and `1`. We can add this function to our `Surreal` class:

```python
    def is_simpler_than(self, other):
        return self.birthday < other.birthday
```

And we can also programatically check this is true:

<iframe width="784" height="128" src="https://datalore.jetbrains.com/view/embed/y0irTQxpwjtJraOPVB5Kuf/11?height=128" frameborder="0"></iframe>

Now, generating these numbers from scratch will quickly get tedius, as on the second day the number of valid Surreals will jump from 3 to 24. We should come up with some sort of combinatoric solution. Skipping slightly ahead in *Grimm*, we can see the numbers generated on the 2nd day are:

```
0 = {|} 1 = {0|} −1 = {|0} {1|}
{−1|} {0, 1|} {0, −1|} {−1, 1|}
{−1, 0, 1|} {|1} {| − 1} {|0, 1}
{|0, −1} {| − 1, 1} {| − 1, 0, 1} {−1|0}
{−1|1} {−1|0, 1} {0|1} {−1, 0|1}.
```

The pattern expressed in the Left and Right sets is known as a **Power Set**. We can use a code recepie from the `itertools` module to simplify their creation.

<iframe width="784" height="192" src="https://datalore.jetbrains.com/view/embed/y0irTQxpwjtJraOPVB5Kuf/12?height=192" frameborder="0"></iframe>

We can then (naievely) generate some Surreals. To save on memory and processing time, we will use an iterator instead of generating all values at once.

<iframe width="784" height="769" src="https://datalore.jetbrains.com/view/embed/y0irTQxpwjtJraOPVB5Kuf/13?height=769" frameborder="0"></iframe>

But what exactly *are* these "numbers" we're creating? We'll explore that, as well as how dyadic rationals (and those beyond) will fit in as well.
