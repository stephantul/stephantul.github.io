---
layout: post
title:  "Getting spacy to only split on spaces."
date:   2019-05-02 00:00:00 +0530
categories: spacy
---
[Spacy](spacy.io) is a fast and easy-to-use python package which can be used to quickly parse, tokenize, tag, chunk across a variety of languages.
I attempted to apply Spacy to data in a NER task for which I had pre-tokenized data with gold standard BIO tags.
Unfortunately, Spacy would still further tokenize some words, while actually I just needed it to split on white space.

The solution to this might not be completely obvious, so I decided to do exactly that.

```python
import spacy
from spacy.tokenizer import Tokenizer

if __name__ == "__main__":

    nlp = spacy.load('en_core_web_sm')
    s1 = nlp("dog's are funny animal's's., haha.__ ")
    nlp.tokenizer = Tokenizer(nlp.vocab)
    s2 = nlp("dog's are funny animal's's., haha.__ ")

    print([token.text for token in s1])
    print([token.text for token in s2])
```

As it turns out, the default arguments for `Tokenizer` amount to selecting no Tokenization at all.

Note that, quite obviously, the tagging accuracy will suffer when you keep noisy tokens. In my case, however, this was necessary to keep the alignment.
