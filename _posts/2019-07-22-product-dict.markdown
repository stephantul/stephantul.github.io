---
layout: post
title:  "Using itertools.product with dictionaries"
date:   2019-07-21-00:00:00 +0530
categories: python
redirect_from:
- python/2019/07/20/product-dict/
---

In a [previous post](https://stephantul.github.io/python/2019/07/20/product/), I talked about using `itertools.product` with lists.
In this post, I used a typical ML experiment as an example, and made a comparison with sklearn's `GridSearchCV`.
It occurred to me that `GridSearchCV` uses dictionaries, while my example only used lists, so in this post I will show you how to build a dictionary iterator using `product`.

First, let's take our basic setting, using the `SVC` from sklearn as an example.
```python
from sklearn.svm import SVC

if __name__ == "__main__":

    params = {"C": [.001, .01, .1],
              "gamma": [.01, 1, 10],
              "kernel": ["linear", "rbf"]}

    for kernel in params["kernel"]:
        bundle = {"kernel": kernel}
        for c in params["C"]:
            bundle["C"] = c
            if kernel == "rbf":
                for gamma in params["gamma"]:
                    bundle["gamma"] = gamma

            clf = SVC(**params)
            # Do some stuff with your clf
```

Now, as you can see, this suffers from the same problems we had before.
This time, however, we can't solve it by using `product`.

```python
from sklearn.svm import SVC
from itertools import product

if __name__ == "__main__":

    params = {"C": [.001, .01, .1],
              "gamma": [.01, 1, 10],
              "kernel": ["linear", "rbf"]}
    z = list(product(*params))
```

If you run the snippet above, you will see that `product` has iterated over the strings in the keys, and has returned the cartesian product over the keys.
This is not what we want.
Even worse, if we happened to have had a non-iterable as a key, such as an integer, product would simply have crashed.

We can also iterate over the values.

```python
from sklearn.svm import SVC
from itertools import product

if __name__ == "__main__":

    params = {"C": [.001, .01, .1],
              "gamma": [.01, 1, 10],
              "kernel": ["linear", "rbf"]}
    keys, values = zip(*params.items())
    for bundle in product(*values):
        d = dict(zip(keys, bundle))
        # This is ugly, but we need a way of saying that we want to skip
        # iterating over gamma if we use a linear kernel.
        if d["kernel"] == "linear" and "gamma" != params["gamma"][0]:
            continue
        clf = SVC(**d)
        # Do something with the classifier.
```

This does what we want. Note that we can't just use `*params.values()` directly, because then we would rely on the dictionaries being in insertion order, which is something we can only rely on [from python 3.6 onwards](https://docs.python.org/3/whatsnew/3.6.html#whatsnew36-compactdict).
This has bitten me at least once, because my own machine ran python 3.6+, while the machine I deployed on ran on 3.5.
It tooke me quite some time to figure out that one!

Finally, in the previous example, remember that we also included the iterations into the product, allowing us to do everything in a single for loop.
This can't be done easily using this format, but with a little bit of extra code it is possible.

```python
from sklearn.svm import SVC
from itertools import product

if __name__ == "__main__":

    params = {"C": [.001, .01, .1],
              "gamma": [.01, 1, 10],
              "kernel": ["linear", "rbf"]}
    # Define the iterations
    iters = range(10)
    keys, values = zip(*params.items())
    for bundle in product(*values, iters):
        # We know the last value of the bundle is the iteration
        # so we drop that value.
        # This is actually unnecessary, because the zip would
        # drop the final argument anyway. But it is clearer!
        d = dict(zip(keys, bundle[:-1]))
        if d["kernel"] == "linear" and "gamma" != params["gamma"][0]:
            continue
        clf = SVC(**d)
        # Do something with the classifier.
```

Here, we use the unpacking operator (`*`), to unpack values, so that it is on the same level as `iters`.
What is cool about this is that we don't actually "loop" over our iterations.
Of course we do everything `iters` times, but we don't actually create a for loop in our code that represents this.
In fact, for reproducible experiments, we could just replace `iters` by 10 random seeds, and then run our experiments 10 (or 100, or 1000) times, without really representing the fact that we are running the algorithm with the same settings.
