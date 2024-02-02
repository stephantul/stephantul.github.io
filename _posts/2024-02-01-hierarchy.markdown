---
layout: post
title:  "Correctly typing recursive hierarchies in Python"
date:   2024-02-01-00:00:00 +0000
categories: python mypy types
---

I recently tried to create a recursive type in Python using mypy. Recursive types naturally occur when processing nested collections of arbitrary depth, such as lists or dictionaries. For me, this most often happens when processing JSON data in the wild.

I found it quite hard to correctly define a recursive type. If you simply want to know how to create one, here is how.

```python
from typing import Union

Hierarchy = list[Union["Hierarchy", str]]

```

This is of course assuming you have nested lists of strings. Your use-case might vary.

# First attempts

In order to explain what is going on, I'll first introduce a function that processes a list of lists of arbitrary depth, and returns all strings from all lists. That is, it flattens the list (I know there are easier ways to flatten lists, bear with me.)

```python
Hierarchy = list[list | str]

def flatten_list(hierarchy: Hierarchy) -> list[str]:
    terminals = []
    for item in hierarchy:
        if isinstance(item, str):
            terminals.append(item)
        else:
            terminals.extend(flatten_list(item))

    return terminals

```

The type here is usually what I would attempt, the assumption being that it is not easy to define a hierarchical type. This example also passes mypy, but this is because the nested `list` is short for `list[Any]`. We can check this by adding `reveal_type` to the code, and re-running `mypy`:

```python
Hierarchy = list[list | str]

def flatten_list(hierarchy: Hierarchy) -> list[str]:
    terminals = []
    reveal_type(hierarchy)
    for item in hierarchy:
        reveal_type(item)
        if isinstance(item, str):
            terminals.append(item)
        else:
            terminals.extend(flatten_list(item))

    return terminals  
  
```

Running `mypy` on this piece of code gives us:

```text
note: Revealed type is "builtins.list[Union[builtins.list[Any], builtins.str]]"
note: Revealed type is "Union[builtins.list[Any], builtins.str]"
```

Which shows that the nested list is typed as `list[Any]`.

# Making it a bit more objective

One way to make the type hierarchical is to introduce a class that has itself as a member. Note that this requires us to import `annotations` from `__future__`.

```python
from __future__ import annotations

from dataclasses import dataclass
from typing import Iterator


@dataclass
class Hierarchy:
    items: list[Hierarchy | str]

    def __iter__(self) -> Iterator[Hierarchy | str]:
        return iter(self.items)

```

This works! It fully type checks and also works in practice. So this means we're done, right? We've created a type that acts as a hierarchical list of items. 

...except we forgot one thing! We haven't actually instantiated these dataclasses. That is, in order to create `Hierarchy` dataclasses, we have to traverse the hierarchy. This would, again, require us to define a hierarchical type to define the code to create the `Hierarchy` objects. One way to fix this, is to create `Hierarchy` objects during traversal. While this works, but I don't really see the point: at that point you're just creating objects to satisfy the type checker.

# Going primitive

In order to solve this, we  need to be able to define a `Hierarchy` in terms of the primitives of the data we ingest. For arbitrary JSON data, and python, this means we need to define a type over `dict`, `list`, and so on.

My first attempt at this was to define something like this:

```python
from __future__ import annotations

Hierarchy = list[Hierarchy | str]
```

Now, what is super confusing about this, is that this correctly passes `mypy` type checking, but fails when actually running the code, and with the following error:

```text
NameError: name 'Hierarchy' is not defined
```

This is probably the first time I saw something that passed static type checks that was simply wrong. Note that `pylance`, which is the default type checker in VSCode, does flag this as an error. Usually, I don't see any discrepancies between the two, so this is interesting. Also note that without `annotations` imported, this also doesn't pass `mypy`.

So, what is happening here is that definitions in Python are built in a bottom-up fashion. In order to define `Hierarchy`, we need to know which types are used by `Hierarchy`. But this is not possible, because `Hierarchy` hasn't been defined at that point.

One way to circumvent this, is by using string aliases instead of the actual type, as follows:

```python
Hierarchy = list["Hierarchy" | str]

```

Note that this no longer requires us to import `annotations`. This does pass type checking, but, again, doesn't actually work! When running this code, it throws a `TypeError`. 

```text
TypeError: unsupported operand type(s) for |: 'str' and 'type'
```

What is happening here? As it turns out, when type checking, `mypy` is happy to pretend that `"Hierarchy"` actually refers to the type `Hierarchy` which we defined earlier on the same line. This allows for recursive references. But at run-time, `"Hierarchy"` no longer represents a type, but is just a good old string. So, what we're actually doing is the following:

```python
result = "Hierarchy" | str
```

I.e., we're using the pipe operator, often used in boolean logic, and apply it to an instance of a `str` and a `type` (which happens to be `str`).
Arbitrary classes can implement their own logic for the pipe operator by implementing the dunder method `__or__`. `str` doesn't implement `__or__`, so using `|` doesn't make a lot of sense. Note that this can actually give a *really really* ugly bug if you do use a type that implements `__or__`, as it won't crash, but will instead return the value of whatever `__or__` returns.

So, given that this is a syntax error, our last recourse could be to introduce good old `Union` from typing. From python 3.10 onwards, the `|` is an alias for `Union`, and I've never imported `Union` since. This is the first time (lots of first times here), that I've seen a replacement for a typing operator not being equal to that typing operator, so, this threw me for a loop again. In fact, I only solved it by reading through the comments of [this SO answer](https://stackoverflow.com/questions/53845024/defining-a-recursive-type-hint-in-python).

So, circling back to the top of this post, the correct answer is to use `typing.Union`.

```python
from typing import Union

Hierarchy = list[Union["Hierarchy", str]]

```

The revealed types in the function above, are:

```text
note: Revealed type is "builtins.list[Union[builtins.str, ...]]"
note: Revealed type is "Union[builtins.str, builtins.list[Union[builtins.str, ...]]]"
```

Which is exactly what we expected. Nice!

