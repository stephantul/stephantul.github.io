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

This model clearly communicates that `data` can either be a single `dict`, or a list of `dict`s. This pattern is common when getting data from a database, i.e., using `mongodb`'s `find` and `findOne`. So far so good!

When we try to parse some things, however, something weird happens:

```python
MyModel.parse_obj({"data": [{"a": "x", "b": "y"}]})
# Output: MyModel(data={'a': 'b'})
MyModel.parse_obj({"data": [{"a": "x", "b": "y", "c": "z"}]})
# Output: MyModel(data=[{'a': 'x', 'b': 'y', 'c': 'z'}])
```

So, despite both input dictionaries having the same structure, they get parsed completely differently. What is going on? Do you see why? Please take a moment to collect your thoughts, and then read on.

# Solution

As seen above, `pydantic` models allow you to specify union types as types for your attributes. As `pydantic` also allows you to automatically parse incoming data, this presents a little bit of an issue, because it could be that a given incoming value is valid for all subtypes of whatever union type you have.

Here's an example:

```python
from pydantic import BaseModel

class MyModel(BaseModel):
    x: str | int

```

If we parse something, we get the following behavior:

```python
MyModel.parse_obj({"x": "snake"})
# Output: MyModel(x='snake')
MyModel.parse_obj({"x": 10})
# Output: MyModel(x='10')

```

This is, again, weird! Somehow, despite us using an integer, this integer gets parsed into a string. By now it might be obvious what is happening: for a given union type, pydantic processes every subtype of the union from left to right. So, in our case, it first tries to coerce the data to `str`, and only if this fails, will it try to parse it as `int`. Because any `int` can also be turned into a `str`, we never actually parse anything as an `int`. The solution here is therefore obvious: reorder your types! 

```python
from pydantic import BaseModel

class MyModel(BaseModel):
    x: int | str

MyModel.parse_obj({"x": "snake"})
# Output: MyModel(x='snake')
MyModel.parse_obj({"x": 10})
# Output: MyModel(x=10)

```

Nice! Now we're also in a nice position to answer the riddle above. Recall that our first input dictionary has the following structure, and that our types were `dict` and `list[dict]`. We'll first try conversion to `dict`:

```python
dict([{"a": "x", "b": "y"}])
# Output: {"a": "b"}!
dict([{"a": "x", "b": "y", "c": "z"}])
# ValueError: dictionary update sequence element #0 has length 3; 2 is required

```

And this shows us our answer! Pydantic first tries to parse the incoming data as a dictionary. Dictionary instantiation with two items is allowed, and will lead to a dictionary with single key-value pair. Because iteration over dictionaries always iterates over the keys, we actually are doing the following:

```python
dict([{"a": "x", "b": "y"}.keys()])
# Which is:
dict([("a", "b")])
# Output: {'a': 'b'}

```

What is interesting, to me, is that this is a very subtle bug: this model could be in production for years, and work correctly for lists of dictionaries with fewer than or more than 2 keys. But if we somehow refactored our code, far in the future, so that dictionaries with exactly 2 keys were ingested, we would suddenly find ourselves with a misbehaving pydantic model: a model that parses the data, but produces an incorrect result. None of this is, of course, the fault of the pydantic developers. 

# Similar issues

A similar issue can pop up when using nested models and using specific `extra` options. Pydantic models allow you to specify what happens to extra data. The following snippet shows what happens.

```python
from pydantic import BaseModel

class MyModel(BaseModel):
    x: dict[str, str]

# ignore
MyModel.parse_obj({"x": {"dog": "animal"}, "y": 10}).dict()
# Output: {"x": {"dog": "animal"}

# allow
MyModel.parse_obj({"x": {"dog": "animal"}, "y": 10}).dict()
# Output: {"x": {"dog": "animal", "y": 10}

# forbid
MyModel.parse_obj({"x": {"dog": "animal"}, "y": 10}).dict()
# ValueError

```

The option `ignore`, which is the default simply ignores your extra keys, while `allow` allows your model to "carry them around" (which is very useful!). `forbid` is super strict, and creates a crash.

Now, consider the following models:

```python
from pydantic import BaseModel

class Name(BaseModel, extra='allow'):
    name: str

class NameAndAge(BaseModel):
    name: str
    age: int

class Base(BaseModel):
    person: Name | NameAndAge

```

Now, if we parse some data, we'll see that we actually run into some really nasty behavior.

```python
Base.parse_obj({"person": {"name": "John"}})
# Output: Base(person=Name(name='John'))
Base.parse_obj({"person": {"name": "John", "age": 10}})
# Output: Base(person=Name(name='John', age=10))

```

As you can see, the second output is actually completely incorrect. This is, again, because pydantic is able to parse the incoming dictionary using the `Name` model, simply because the `Name` model does not reject any superfluous keys, and therefore never fails to parse. 

Now, what is super interesting, is that there is _no_ situation in which `NameAndAge` would be instantiated. In general, it is _never_ possible for the following to work without custom validation:

1. You define a union of types, which are also `BaseModel`s
2. The first of models sets `extra` to `"allow"`
3. The types all share common fields

If you have this situation, and don't implement custom validation, the first type in your union is the only type that will ever get instantiated. Your other type, for all intents and purposes, is inert.
