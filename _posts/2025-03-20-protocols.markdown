---
layout: post
title:  "Protocols to make untyped code behave"
date:   2025-03-20-00:00:00 +0000
categories: python typing
redirect_from:
- python/typing/2025/03/20/protocols/
---

Working with external untyped code in a typed code base can be challenging, you'll get lots of `Any` or `Unknown`, which might propagate through your codebase. This can force you to reach for `typing.cast`, or `# type: ignore` statements, which kind of defeats the purpose of using static typing in the first place.

One easy way to deal with this is by leveraging a [`Protocol`](https://mypy.readthedocs.io/en/stable/protocols.html). I've discussed these before in the context of caching, see [here](https://stephantul.github.io/blog/cache). The gist of it is that a `Protocol` allows you to define a "structure" to be used in place of the thing that is actually used. Anything that matches the implementation of that `Protocol` can be used in place of (implements) the protocol.

One of the most well-known protocols is the `Iterator`. First, let's look at a familiar example:

```python
def prefix_strings(items: list[str], prefix: str) -> list[str]:
    return [f"{prefix}{s}" for s in items]
```

This code _works_ with tuples, sets, dicts, whatever. But it is typed as if it only works with lists. This is better:

```python
from typing import Iterator

def prefix_strings(items: Iterator[str], prefix: str) -> list[str]:
    return [f"{prefix}{s}" for s in items]
```

The `Iterator` protocol only tells us that `items` can be iterated over, but doesn't make any other assumptions This is fine! We're just iterating over it, so we don't need any other functionality.

To recap: you use `Protocol` when you want to accept all objects that implement specific functions, and don't want to restrict yourself to a `TypeVar` with union types. As you can probably surmise from the example above, using a `Protocol` matches how we often use duck typing in python.

# Unbehaved code

Code that is untyped often makes it really difficult to get typing to work. As an example, let's look at the [`tokenizers`](https://github.com/huggingface/tokenizers) library from Hugging Face. The python bindings in tokenizers are, unfortunately, untyped. So every time you tokenize something, you'll have to manually provide types, or live with `Any`. Compounding this, is that `Tokenizer.encode`, which is the function you'll most often use, does not return a python primitive, but another structured object (`Encoding`) which is untyped as well.

Here's an example:

```python
from tokenizers import Tokenizer, Encoding

tokenizer = Tokenizer.from_pretrained("bert-base-uncased")

tokens = tokenizer.encode("dog")
reveal_type(tokens)
# note: Revealed type is "Any"
ids = tokens.ids
reveal_type(ids)
# note: Revealed type is "Any"
```

So, in this case, we could try to do this:

```python
from tokenizers import Tokenizer, Encoding

tokenizer = Tokenizer.from_pretrained("bert-base-uncased")

tokens = cast(Encoding, tokenizer.encode("dog"))
reveal_type(tokens)
# note: Revealed type is "Any"
ids = tokens.ids
reveal_type(ids)
# note: Revealed type is "Any"
```

This still makes `mypy` type both variables as `Any`. Even if it got the first one correct, it would still trip over the second one, because it would not know the type of `Encoding.ids` is `list[int]`.

Here's how to solve this using a minimal `Protocol`.

```python
from typing import Protocol

class EncodingProtocol(Protocol):

    def ids(self) -> list[int]:
        ...

class TokenizerProtocol(Protocol):

    def encode(self, sequence: str, 
               *args: Any, 
               **kwargs: Any) -> EncodingProtocol:
        ...

tokenizer: TokenizerProtocol = Tokenizer.from_pretrained("bert-base-uncased")

tokens = tokenizer.encode("dog")
reveal_type(tokens)
# note: Revealed type is "EncodingProtocol"
ids = tokens.ids
reveal_type(ids)
# note: Revealed type is "builtins.list[builtins.int]"
```

As you can see, using a `Protocol` has a couple of advantages. First, if you type the protocol correctly, you get a cascading effect. Now, wherever we pass a `Tokenizer` to a function, we can just pass a `TokenizerProtocol`, and any code that calls `encode` will then return an `Encoding`, which is also properly typed. Second, we didn't need to provide types for anything beyond the things we actually use. `Tokenizer` and `Encoding` have many functions, helpers, etc. that we don't need to describe for our specific purpose. Using a protocol thus gets you a nice, minimal, way of describing exactly what you need from the external library in terms of typing.

# Downsides

One of the biggest downsides to using a protocol is, ironically, exactly the same one as using `cast` or `# type: ignore` markers: if the underlying implementation ever changes, or your protocol is defined incorrectly, the static analyzer will be confidently incorrect. This also reminds me of arguments against docstrings: it is better to have no docstrings than incorrect docstrings. Similarly, it is better to have no typing than to have incorrect typing.

Note that these errors can be quite subtle. For example, consider that, maybe, when presented with an empty string, our tokenizer would return `None`. This is difficult to figure out without looking at the actual implementation, but you'd still have to deal with it somehow.

Second, and I think this is a small objection, is that it simply takes a bit of time to write the `Protocol`. I don't think this is actually a valid objection. If you allow me the slippery slope argument: if you think writing code for consistent correct types is too much work, then why write types at all?

# Alternatives

A nice alternative is to accept that the tokenizer returns an untyped thing, and to just wrap it in another function.

```python
def string_to_ids(string: str, tokenizer: Tokenizer) -> list[int]:
    return cast(list[int], tokenizer.encode(string).ids)
```

This also has the effect of encapsulating the untyped code, and has the added advantage of also abstracting away from the specific function used. So if that is also your goal, you could consider doing this instead. It does require a cast to work, so that's an unfortunate side effect.

# Conclusion

Writing well-typed code makes you faster in the short and in the long run, and reduces the probability of catastrophic failure. I think using a `Protocol` to provide types makes you much more confident about what you are using these tools for.
