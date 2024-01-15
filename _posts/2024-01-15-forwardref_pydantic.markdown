---
layout: post
title:  "New year's resolutions"
date:   2024-01-15-00:00:00 +0000
categories: python pydantic
---

I recently ran into the following error when initializing a Pydantic `BaseModel`:

```
ConfigError: field "other_model" not yet prepared so type is still a ForwardRef, you might need to call Model.update_forward_refs().
```

This threw me for a loop, Googling didn't really reveal anything useful, except [this SO post](https://stackoverflow.com/questions/71510622/pydantics-update-forward-refs-raises-typing-nameerror), which doesn't give the correct answer. So I decided to dig in!

First off, the file I was importing `Model` from looked a bit like this. It consists of a `BaseModel` that has another `BaseModel` as a field. 

```python
from __future__ import annotations

from pydantic import BaseModel

class Model(BaseModel)
    other_model: OtherModel

class OtherModel(BaseModel)
	name: str
```

Calling `Model.update_forward_refs()` solved the problem for me, but it weirded me out a bit. Why was this necessary all of a sudden? And why didn't I see this error before?

Well, as it turns out, this is a weird interaction between the `__future__` import and pydantic. From `__future__ import annotations` does a _LOT_ of things, but one of the things it does is that it allows you to define classes in any order. Normally, Python classes can only be defined from top to bottom: a class that is referenced by another class needs to be defined on a lower line number than the class that references it. By importing `annotations`, this limitation is removed. pydantic can't deal with this limitation, and thus throws an error.

The real solution to this error is thus not to call `update_forward_refs()` (although that solves it!), but to define your classes in the order required by python as if `annotations`wasn't imported. Rewriting the above to:

```python
from __future__ import annotations

from pydantic import BaseModel

class OtherModel(BaseModel)
	name: str

class Model(BaseModel)
    other_model: OtherModel

```

solves the issue just fine. 