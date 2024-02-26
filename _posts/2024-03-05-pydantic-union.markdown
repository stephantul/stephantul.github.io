---
layout: post
title:  "Pydantic puzzle"
date:   2024-03-02-00:00:00 +0000
categories: python pydantic
---

This post will assume you know a bit about [`pydantic`](https://pydantic.dev/), and also a little bit about Python typing. If you think you feel comfortable with the above, here's an interesting issue I came across when parsing incoming data in `pydantic`:

Consider the following model:

```python
from typing import Any

from pydantic import BaseModel

class MyModel(BaseModel):
    data: dict[str, Any] | list[dict[str, Any]]

```

This model clearly communicates that `data` can either be a single `dict`, or a list of `dict`s. This pattern, although a bit of an antipattern, occurs when getting data from a database, i.e., using `mongodb`'s `find` and `findOne`. So far so good!

When we try to parse some things, however, something weird happens:

```python
MyModel.model_validate({"data": [{"a": "x", "b": "y"}]})
# Output: MyModel(data={'a': 'b'})
MyModel.model_validate({"data": [{"a": "x", "b": "y", "c": "z"}]})
# Output: MyModel(data=[{'a': 'x', 'b': 'y', 'c': 'z'}])
```

We expected both of them to be parsed as lists of dictionaries, but that's not what is happening. What is going on? Do you see why? Please take a moment to think of why this is happening, and then read on.

...

...

(keep thinking)

...

...

(did you have something nice for dinner last night?)

...

...

# Solution

Now, the solution! But, I wouldn't be me if I first tried to explain what was going on. As seen above, `pydantic` models allow you to specify union types as types for your attributes. As `pydantic` also allows you to automatically parse incoming data, this presents a little bit of an issue, because it could be that a given incoming value is valid for all subtypes of whatever union type you have.

Here's an example:

```python
from pydantic import BaseModel

class MyModel(BaseModel):
    x: str | int

```

If we parse something, we get the following behavior:

```python
MyModel.model_validate({"x": "snake"})
# Output: MyModel(x='snake')
MyModel.model_validate({"x": 10})
# Output: MyModel(x='10')

```

This is, again, weird! Somehow, despite us using an integer, and one of the types literally being an integer, this integer gets parsed into a string. By now it might be obvious what is happening: for a given union type, pydantic processes every subtype of the union from left to right, and simply uses the first type fits. This algorithm is shown below in extremely ugly pseudo-code:

```python
def parse_data_with_union(data: Any, union: list[type]):
    for possible_type in union:
        try:
            return possible_type(data)
        except Exception:  # Don't do this IRL.
            continue
    
    raise ParsingError()  # Or something

```

I foolishly assumed that, for a given union, pydantic would try all options, and pick the best one, using type hierarchies or something. 

So, to be overly verbose, in the example above, the model first tries to coerce the data to `str`, and only if this fails, will it try to parse it as `int`. Because any `int` can also be turned into a `str`, we never actually parse anything as an `int`. The solution here is therefore obvious: reorder your types! 

```python
from pydantic import BaseModel

class MyModel(BaseModel):
    x: int | str

MyModel.model_validate({"x": "snake"})
# Output: MyModel(x='snake')
MyModel.model_validate({"x": 10})
# Output: MyModel(x=10)

```

Nice! Now we're also in the position to answer the riddle above. Recall that our first input dictionary has the following structure, and that our types were `dict` and `list[dict]`. We'll first try conversion to `dict`:

```python
dict([{"a": "x", "b": "y"}])
# Output: {"a": "b"}!
dict([{"a": "x", "b": "y", "c": "z"}])
# ValueError: dictionary update sequence element #0 has length 3; 2 is required

```

And this gives us the answer. Pydantic first tries to parse the incoming data as a dictionary. Dictionary instantiation with two items is allowed, and will lead to a dictionary with single key-value pair. Because iteration over dictionaries always iterates over the keys, we actually are doing the following:

```python
dict([{"a": "x", "b": "y"}.keys()])
# Which is:
dict([("a", "b")])
# Output: {'a': 'b'}

```

What is interesting, to me, is that this is an extremely subtle bug: this model could be used in a production API or queue for years, and work correctly for lists of dictionaries with fewer than or more than 2 keys. But if we somehow changed our datamodel, far in the future, so that dictionaries with exactly 2 keys were ingested, we would suddenly find ourselves with a misbehaving pydantic model: a model that parses the data, but produces an incorrect result. None of this is, of course, the fault of the pydantic developers. 

# Similar issues

A similar issue can pop up when using nested models and using specific `extra` options. Pydantic models allow you to specify what happens to extra data. The following snippet shows what happens.

```python
from pydantic import BaseModel

class MyModel(BaseModel):
    x: dict[str, str]

# ignore
MyModel.model_validate({"x": {"dog": "animal"}, "y": 10}).dict()
# Output: {"x": {"dog": "animal"}

# allow
MyModel.model_validate({"x": {"dog": "animal"}, "y": 10}).dict()
# Output: {"x": {"dog": "animal", "y": 10}

# forbid
MyModel.model_validate({"x": {"dog": "animal"}, "y": 10}).dict()
# ValueError

```

The option `ignore`, which is the default, simply ignores your extra keys, while `allow` allows your model to "carry them around" (which is very useful!). `forbid` is more strict, and crashes when extra data is passed.

Now, consider the following models:

```python
from pydantic import BaseModel

class Name(BaseModel):
    name: str

class NameAndAge(BaseModel):
    name: str
    age: int

class Base(BaseModel):
    person: Name | NameAndAge

```

Now, if we parse some data, we'll see that we actually run into some really nasty behavior.

```python
Base.model_validate({"person": {"name": "John"}})
# Output: Base(person=Name(name='John'))
Base.model_validate({"person": {"name": "John", "age": 10}})
# Output: Base(person=Name(name='John', age=10))

```

As you can see, the second output is actually incorrect. This is, again, because pydantic is able to parse the incoming dictionary using the `Name` model, simply because the `Name` model does not reject any superfluous keys, and therefore never fails to parse. 

Now, what is super interesting, is that there is _no_ situation in which `NameAndAge` would be instantiated. In general, it is _never_ possible for the following to work without custom validation:

1. You define a union of types, which are also `BaseModel`s
2. The fields of a prior model are a subset of a later model
3. You don't set `extra` to `"ignore"`

# None

One exception to what I wrote above, is our good friend `None`. `pydantic` ignores the usual precedence rules for `None`, and matches `None` regardless of where in union it is located. So, unlike with other types, the following two models are equivalent:

```python
from pydantic import BaseModel

class Model(BaseModel):
    a: None | str
    
class Model2(BaseModel):
    a: str | None

Model.model_validate({"a": None})
# Output: Model(a=None)
Model2.model_validate({"a": None})
# Output: Model2(a=None)

```

The reasoning behind this is probaly that `str | None` means that we either specify a string, or don't specify anything (i.e., the field is optional). `None` is also a singleton value in python, which makes it easy to work with and check for.

And that's it! I hope you enjoyed this puzzle and learned something useful.