---
layout: post
title:  "Protocols to make untyped code behave"
date:   2025-03-21-00:00:00 +0000
categories: python typing
---

Working with untyped code in a typed code base can be challenging, you'll get lots of `Any` or `Unknown`, which might propagate through your codebase. This can force you to reach for `typing.cast`, or ignored types, which kind of defeats the purpose of using static typing in the first place.

One easy way to deal with this is by leveraging a [`Protocol`](https://mypy.readthedocs.io/en/stable/protocols.html). I've discussed these before in the context of caching, see [here](https://stephantul.github.io/python/typing/2024/05/31/cache/). The gist of it is that a `Protocol` allows you to define a "structure" to be used in place of the thing that is actually used. Anything that implements all functions that the protocol implements can be used in the place where te protocol is used.

See here for an example:

```python
def prefix_strings(strings: list[str], prefix: str) -> list[str]:
    return [f"{prefix}{s} for s in strings]
```

This code _works_ with tuples, sets, dicts, whatever. But it is typed as if it only works with lists. This is much better:

```python
from typing import Iterator

def prefix_strings(strings: Iterator[str], prefix: str) -> list[str]:
    return [f"{prefix}{s} for s in strings]
```

You use `Iterator` when you want to not be tied to a specific implementation, and don't want to restrict yourself to a `TypeVar` with union types. As you can probably surmise from the example above, using a `Protocol` matches how we often use duck typing in python.

# Unbehaved code

Code that is untyped often makes it really difficult to get typing done. As an example, let's look at the [`tokenizers`](https://github.com/huggingface/tokenizers) library from hugging face. The python bindings in tokenizers are, unfortunately, untyped. So every time you tokenize something, you'll have to manually type it. What's worse, is that the library does not return a python primitive, but another structured object, `Encoding`, which is also untyped.

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

This still makes `mypy` type both variables as `Any`. Even if it got the first one correct, it would still trip over the second one, because it would not know that `Encoding` had an attribute `ids`, and hence would also not have a type for this attribute.

A way out is to use a minimal `Protocol`.

```python
from typing import Protocol

class Encoding(Protocol):

    def ids(self) -> list[int]:
        ...

class TokenizerProtocol(Protocol):

    def encode(self, text: str | list[str], *args: Any, **kwargs: Any) -> Encoding:
        ...

tokenizer: TokenizerProtocol = Tokenizer.from_pretrained("bert-base-uncased")

tokens = cast(Encoding, tokenizer.encode("dog"))
reveal_type(tokens)
# note: Revealed type is "Encoding"
ids = tokens.ids
reveal_type(ids)
# note: Revealed type is "builtins.list[builtins.int]"
```

As you can see, using a `Protocol` has a couple of advantages. First, if you type the protocol correctly, you get a cascading effect. Now, wherever we pass a `Tokenizer` to a function, we can just pass a `TokenizerProtocol`, and any code that calls `encode` will then return an `Encoding`, which is also properly typed. Second, we didn't need to implement anything beyond what we actually used. `Tokenizer` and `Encoding` have many functions, helpers, etc. that we don't need to describe for our specific purpose. Using a protocol thus gets you a nice, minimal, way of describing exactly what you need from the external library in terms of typing.

# Downsides

One of the biggest downsides to using a protocol is, ironically, exactly the same one as using `cast` or `# type: ignore` markers: if the underlying implementation ever changes, or your protocol is defined incorrectly, the static analyzer will be confidently incorrect. This also reminds me of arguments against docstrings: sometimes it is better to have no docstrings than incorrect docstrings.

Note that the errors can be quite subtle. For example, consider that, maybe, when presented with an empty string, our tokenizer would return `None`. This is difficult to figure out without looking at the actual implementation, but you'd still have to deal with it somehow.

Second, and I think this is a small objection, is that it simply takes a bit of time to write the `Protocol`. I don't think this is actually a valid objection. If you allow me the slippery slope argument: if you think writing code for consistent correct types is too much work, then why write types at all?

# Conclusion

Finally, I would like to end on a cautionary note. On my last post, some person I don't know insinuated that this type of content is outdated, impractical and not suited for "real programmers" who prioritize working code over beautiful code. The implicature being that I am a programmer who lives in a glass castle, and not in the mines, where the real programmers do their work. Buddy, have I got news for you! Not only am I in the mines, I am the king of the mines. I am a dwarf! Literally no one I ever worked with would accuse me of living in a glass castle. I would destroy it with my axe and blundering bravado!

On a more serious note, I obviously don't agree with this sentiment: writing well-typed code makes you faster in the short and in the long run, and reduces the probability of catastrophic failure. I am also writing this because I think it is fun, not because I think it is necessary. So, the next time you feel like writing snarky comments to someone who just likes writing about types, please keep this in mind and go do something more productive.
