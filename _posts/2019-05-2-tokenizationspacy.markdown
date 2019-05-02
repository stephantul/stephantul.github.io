---
layout: post
title:  "Getting spacy to only split on spaces."
date:   2019-05-02 00:00:00 +0530
categories: spacy
---
[Spacy](https://spacy.io) is a fast and easy-to-use python package which can be used to quickly parse, tokenize, tag, and chunk with high accuracy across a variety of languages.
I attempted to apply `spacy` to a NER task for which I had pre-tokenized data with gold standard BIO tags.
Unfortunately, the default pipeline would still further tokenize some words, while actually I just needed it to split on space characters.

The solution to this is not completely obvious, so here's a snippet that allows a standard `spacy` pipeline to just split on space.

```python
import spacy
from spacy.tokenizer import Tokenizer

if __name__ == "__main__":

    nlp = spacy.load('en_core_web_sm')
    s1 = nlp("dog's are funny animal's's._ ")
    nlp.tokenizer = Tokenizer(nlp.vocab)
    s2 = nlp("dog's are funny animal's's._ ")

    print([token.text for token in s1])
    # ['dog', "'s", 'are', 'funny', 'animal', "'s", "'s", '.', '_']
    print([token.text for token in s2])
    # ["dog's", 'are', 'funny', "animal's's._"]
```

As it turns out, the default arguments for `Tokenizer` amount to only splitting on space characters.
Note that all the other components in the pipeline are maintained, although keep in mind that their accuracy will suffer because they might get unexpected tokens.
In my case, however, this was the only way to proceed.

Note that this fails for tokens that are separated by more than one space character (thanks to my colleague Madhumita Sushil for pointing this out.)

```python
    s3 = nlp("dog's    are funny     haaa.")
    print([token.text for token in s3])
    # ["dog's", '   ', 'are', 'funny', '    ', 'haaa.']
```

This can be remedied by removing the double spaces before passing them to `spacy`.
