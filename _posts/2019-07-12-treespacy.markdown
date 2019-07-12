---
layout: post
title:  "Recursively dealing with trees"
date:   2019-07-12 00:00:00 +0530
categories: trees
---
For a project, I wanted to do some work on composition over trees.
A typical way to process a tree is to build a recursive function, i.e., a function that calls itself.
The motivation behind using a recursive function is that all non-terminal nodes in a tree are themselves trees.

Below is a simple example of such a recursive function:

```python
def recursive_tree(tree):
    """
    Recursively do nothing.

    Parameters
    ----------
    tree : list (of list)* of strings
        A tree represented as lists. The terminals are assumed to be strings.
        Example: (("dog", "cat"), "walked")

    """
    if not isinstance(tree, (list, tuple)):
        return "VISITED-{}".format(tree)
    res = []
    for x in tree:
        res.append(recursive_tree(x))
    return res
```

Here's a more useful example:

```python
import numpy as np

def recursive_compose(tree, embeddings, emb_size):
    """
    Recursively compose a tree.

    If a word is OOV (not in embeddings), we replace it with a zero vector.

    Parameters
    ----------
    tree : list (of list)* of strings
        A tree represented as lists. The terminals are assumed to be strings.
        Example: (("dog", "cat"), "walked")
    embeddings : dict
        A dictionary mapping from words to their word vectors.
    emb_size : int
        The embedding size.

    """
    if not isinstance(tree, (list, tuple)):
        return embeddings.get(tree, np.zeros(emb_size))
    res = []
    for x in tree:
        res.append(recursive_compose(x, embeddings, emb_size))
    return np.mean(res, 0)
```

This function recursively composes a parse tree applying the mean function for the word embeddings in some sentence.
A similar mechanism has been used as a baseline in [Socher et al. 2011](http://papers.nips.cc/paper/4204-dynamic-pooling-and-unfolding-recursive-autoencoders-for-paraphrase-detection.pdf), although they used binary parse trees.

So, there you have it, some simple functions to deal with parse trees represented as lists of lists.
