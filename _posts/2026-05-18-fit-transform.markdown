---
layout: post
title:  "Scikit-learn's fit transform paradigm is probably not for you"
date:   2026-05-17-00:00:00 +0000
categories: 
    - python
---

If you've ever used code from [scikit-learn](https://scikit-learn.org/stable/index.html), you will have seen the following pattern:

```python
import numpy as np

from sklearn.preprocessing import StandardScaler

X = np.random.randn((100, 32))

scaler = StandardScaler()
scaler.fit(X)
X_transformed = scaler.transform(X)

# Or equivalently
X_transformed = scaler.fit_transform(X)
```

For all `scikit-learn` transformers ([^1]), the `fit` call sets the internal state of the object, while the `transform` call uses the set internal state to transform some data into something else. ([^2]) This paradigm is really useful because it allows for zero-cost chaining: any sequence of transformations can be `fit_transform`ed by simply calling `fit_transform` on all transformations in sequence.

## Conflation between construction and usage

The main point I'll be making in this article is that scikit-learn's fit transform paradigm mixes up the factory pattern, that is, an object that instantiates other objects, with the actual objects. This is used really well by scikit-learn, but probably doesn't fit your codebase.

To illustrate, let's reimplement the `StandardScaler` using numpy: ([^3])

```python
from __future__ import annotations

import numpy as np

class StandardScaler:

    def __init__(self, with_mean: bool = True, with_std: bool = True) -> None:
        self.mean: None | np.ndarray = None
        self.std: None | np.ndarray = None
        self.with_mean = with_mean
        self.with_std = with_std

    def fit(self, X: np.ndarray) -> StandardScaler:
        if self.with_mean:
            self.mean = X.mean(0)
        if self.with_std:
            self.std = X.std(0)
        
        return self

    @property
    def _is_fit(self) -> bool:
        if self.with_mean and self.mean is None:
            return False
        if self.with_std and self.std is None:
            return False
        return True

    def transform(self, X: np.ndarray) -> np.ndarray:
        if not self._is_fit:
            raise ValueError("Standardscaler has not been fit")
        if self.with_mean:
            X = X - self.mean
        if self.with_std:
            X = X / self.std
        return X

    def fit_transform(self, X: np.ndarray) -> np.ndarray:
        self.fit(X)
        return self.transform(X)

```

Let's first talk about the initializer. In a scikit-learn initializer, you are only supposed to set the so-called _hyperparameters_ of a transformer or estimator.That is, you should only set attribues that do not depend on the data you will use to fit the model. So, in this case, the parameters of the initializer determine what the behavior of the instantiated `StandardScaler` will be. So, in our case, `with_mean` and `with_std` determine what the behavior is of the `StandardScaler` that is _produced_ by fitting the `StandardScaler` on some data; if we set `with_mean` to `False`, we actually get a different object than we would get if we set it to `True`.

Second, note that the `fit` function is destructive. It erases the original state, and introduces a completely new state. From a python perspective, however, the same _object_ is returned, its only the internal state that is reset.

Third, note that there is no need to store the hyperparameters once you've fit the transformer. [^4]

Fourth, for a given `StandardScaler`, it is impossible to know whether it has been fit or not. So, whenever you work with `scikit-learn`'s internals, you'll have to continuously check whether the estimators and transformers you work with actually have their internal state set. 

Fifth, when you write your own transformers and estimators, it is very easy to incorrectly implement this state. ([^5])

## Splitting out the factory

So, now on to my main thesis: this whole problem can be avoided by conceding that `StandardScaler` is both a factory and the object that is constructed by the factory. As such, if we split this up into two separate classes, we'll see that we'll end up with much cleaner code.

```python
from __future__ import annotations

import numpy as np

class StandardScaler:

    def __init__(self, mean: np.ndarray | None, std: np.ndarray | None) -> None:
        self.mean: np.ndarray | None = mean
        self.std: np.ndarray | None = std

    def transform(self, X: np.ndarray) -> np.ndarray:
        if self.mean is not None:
            X = X - self.mean
        if self.std is not None:
            X = X / self.std
        return X

class StandardScalerFactory:

    def __init__(self, with_mean: bool = True, with_std: bool = True) -> None:
        self.with_mean = with_mean
        self.with_std = with_std

    def fit(self, X: np.ndarray) -> StandardScaler:
        mean, std = None, None
        if self.with_mean:
            mean = X.mean(0)
        if self.with_std:
            std = X.std(0)
        
        return StandardScaler(mean, std)

    def fit_transform(self, X: np.ndarray) -> tuple[StandardScaler, np.ndarray]:
        scaler = self.fit(X)
        return scaler, scaler.transform(X)

```

As you can see, we've changed the structure considerably. `fit` now returns an object which implements `transform`, and only implements `transform`. `fit_transform` returns a tuple, the first item of which is the fit object, the second of which is the transformed data. This still allows us to forward state in a single call as follows:

```python
transformers = [...]  # Some list of transformers
X = ...  # some numpy array:

fit_transformers = []
for transformer in transformers:
    fit_transformer, X = transformer.fit_transformer(X)
    fit_transformers.append(fit_transformer)

```

So what did we gain? A couple of things:

1) We can guarantee that the object we're dealing with has been fit on some data, and is usable.
2) We clearly separate between creation (the factory) and usage.
3) We have much fewer checks

The main advantage to this is that we have very strong typing guarantees. For every `fit`, we can statically detect what the type object is, and whether it is usable to transform and predict. For example, with base classes:

```python
from typing import Generic

class BaseTransformer:

    def transform(self, X: np.ndarray) -> np.ndarray:
        ...

T = TypeVar("T", BaseTransformer)

class BaseFactory(Generic[T]):

    def fit(self, X: np.ndarray) -> T:
        ...

```

One downside of this pattern is that the hyperparameters are no longer accessible on the fit object. 

In a follow-up post, we'll investigate how we can improve on this pattern and have our cake and eat it to.

### Footnotes

[^1]: A transformer here is something that transforms some data, not a transformer in the machine learning sense.

[^2]: scikit-learn also implements predictors, which have `fit`, `predict` and `fit_predict` functions.

[^3]: In a serious implementation, we'd derive from a base class, use generics, etc.

[^4]: Although doing so is very useful for reproducing research.

[^5]: I don't think this is a problem of `scikit-learn` itself though. Their estimators are all implemented correctly. This is easy to get wrong, however.
