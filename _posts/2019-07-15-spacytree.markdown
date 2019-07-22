---
layout: post
title:  "Parsing a spacy json to a tree"
date:   2019-07-14 00:00:00 +0530
categories: trees, spacy
---
In a [previous post](https://stephantul.github.io/trees/2019/07/10/treerec/), I discussed some easy functions to recursively deal with trees, which often comes up in NLP when trying to deal with parse trees.

By far the easiest way to obtain good (for English at least) parse trees is [spacy](https://spacy.io). One thing that surprised me, however, is that spacy only offers a `.to_json` function that turns a parsed sentence into a flat JSON representation, which looks like this:

```
{"ents": [], # list of entities
 "text": "", # raw text
 "sents": [{"begin": int, "end":int}], # Sentence spans in text
 "tokens": [{"id": int,
             "head": int,
             "dep": str,
             ...}, {...}]} # Token information.
```

There are at least two problems with using this directly.
First, the JSON can contain multiple sentences, and thus multiple parse trees.
Second, the tokens do not contain pointers to their children, making direct traversal non-trivial.

To counteract these two issues, I wrote two functions that turn the JSON to a tree.

```python
from collections import defaultdict

def encode_sent(j, skip):
    """Encode a single sentence."""
    new_dict = defaultdict(dict)
    for x in j["tokens"]:
        s, e = x['start'], x['end']

        stripped = {k: v for k, v in x.items()
                    if k and k not in skip}
        stripped["text"] = j["text"][s:e]
        new_dict[x["id"]].update(stripped)
        if x["dep"] == "ROOT":
            root = new_dict[x["id"]]
        elif x["dep"]:
            # We don't use defaultdicts because we would then
            # return defaultdicts of defaultdicts, which is undesirable.
            try:
                new_dict[x["head"]][x["dep"]].append(new_dict[x["id"]])
            except KeyError:
                new_dict[x["head"]][x["dep"]] = [new_dict[x["id"]]]

    return root


def tree_to_dict(j, skip=None):
    if skip is None:
        skip = set()
    else:
        skip = set(skip)

    for idx, x in enumerate(j['sents']):
        new_j = {}
        s, e = x['start'], x['end']
        new_j["tokens"] = []
        for tok in j["tokens"]:
            # end characters are inclusive.
            if tok["start"] >= s and tok["end"] <= e:
                # Align the tokens with the new text
                tok["end"] -= s
                tok["start"] -= s
                new_j["tokens"].append(tok)
        new_j['text'] = j['text'][s:e]
        yield encode_sent(new_j, skip)
```

`tree_to_dict` takes as input the JSON generated from `.to_json` and separates all the tokens by sentence.
Each sentence is then processed separately by `encode_sent`.

`encode_sent` simply adds all tokens as entries to their parents, and finally returns the root node.
Since all tokens in the sentence except the root node are attached to other tokens, returning the root node ensures that we can access all tokens by recursively traversing the root node.
A peculiarity of the fact that we are using `dict` to store our tree is that we need to create a key, `text`, to store the words themselves.

The `skip` argument of both functions is a way of telling the function which information to omit. If you are not interested in `pos` information in your final tree, you can tell the parser to omit this information.

This function can be directly used on a spacy sentence.


```python
import spacy

# Or another model of your choosing
s = spacy.load("en_core_web_sm")
sentence = s("The dog walked home. His faucet was running, and he had to call an expensive plumber.")

tree = tuple(tree_to_dict(sentence.to_json()))

print(len(tree)) # 2 sentences
>>> 2
print(tree[0].keys())
>>> ['nsubj', 'id', 'start', 'end', 'pos', 'tag', 'dep', 'head', 'text', 'advmod', 'punct']
print(tree[1].keys())
>>> ['nsubj', 'aux', 'id', 'start', 'end', 'pos', 'tag', 'dep', 'head', 'text', 'punct', 'cc', 'conj']
```

You can see that the root nodes of the trees have keys corresponding to their dependency tags.

Finally, note that `tree_to_dict` uses a `yield` instead of a `return`.
This causes the function to define a generator instead of returning values.
In addition to saving us three lines of code, this creates a more flexible interface for the caller of the function, as it is now the caller instead of the function that determines what the output of the function will look like.
If the caller wants a list, they can just do `list(tree_to_dict(json_file))`. If they want a tuple, they can do `tuple(tree_to_dict(json_file))`.
This also has the advantage of the function not computing the results until it is needed. If a user only ever wants the first sentence, they can use a `next` call.
This is what is known as the `faucet pattern`.
