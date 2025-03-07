---
layout: post
title:  "Exposing string types for your users' happiness"
date:   2025-03-07-00:00:00 +0000
categories: python typing
---

Regular users of my blog will know that I am opposed to what is known as [stringly typing](https://wiki.c2.com/?StringlyTyped): using strings in place of more strongly typed identifiers. As an example, consider a language-specific tokenizer:

```python
encoded = tokenizer(["The dog walks"], language="en")
```

What are all the possible values of the language variable, and what do they mean? It's difficult to figure out without diving into the docs or the code itself. A nice workaround would have been to use an `Enum`:

```python
from enum import Enum

class Language(Enum):
    EN = 1
    FR = 2

encoded = tokenizer(["The dog walks"], language=Language.EN)
```

This changes the type of `language` from `str` to `Language`. Using an `Enum` here has an unfortunate side effect: your users will need to import the `Enum` for this to work, which can be painful for new users. So, a nice work-around is to use an enum with string members.

```python
from enum import Enum

class Language(Enum):
    EN = "en"
    FR = "fr"
```

The string members are important! When moving to an `Enum`, `language` will get the following type:

```
language: Language | str
```

And in the tokenizer code, you would have something like this:

```python
selected_language = Language(language)
```

Because calling an `Enum` accepts both members and values of `Language` objects, this function allows `tokenize` to work both with enum input, and string input. The calls below are equivalent.

```python
encoded = tokenizer(["The dog walks"], language="en")
encoded = tokenizer(["The dog walks"], language=Language.EN)
```

because:

```python
Language(Language.EN) == Language("en")
```

In addition to this, your users get a nice error message when they pass a string that isn't a member of `Language`:

```python
lang = Language("de")
# ValueError: 'de' is not a valid Language
```

So moving from a string to an `Enum` has very nice usability benefits: your internal implementation gets a nice boost because you lose all internal string references, at very little cost to your external users. This also makes your code more maintainable: if this function is called by other parts of your code, you can just use `Language` instead of relying on strings. Cool!

In addition, you can easily update your error messages and docs, for example:

```python
try:
    selected_language = Language(language)
except ValueError:
    language_names = ", ".join([lang.name for lang in Language])
    logger.error(f"Invalid languages. Valid languages are: {language_names})
```

If you had used strings, this would also need to be kept up to date somehow, possibly through some external variable.

Finally, one interesting tidbit here is the transformers library. The tokenizer in the transformers library has a function called `encode_plus`, which has a variable named `return_tensors`. This function exactly follows the advice outlined above: it takes both `str` and `Enum` members as input (`transformers.utils.generic.TensorType`). Yet, everyone just uses the string version. I've never seen transformers code actually import and use the enum. So, even if you supply this functionality to your users to make them happy, they will maybe not ever use it, or find out about it.
