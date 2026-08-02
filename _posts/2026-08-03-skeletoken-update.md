---
layout: post
title:  "Skeletoken 0.4.0 release"
date:   2026-08-02-00:00:00 +0000
categories: tokenization 
---

[`skeletoken`](https://github.com/stephantul/skeletoken) has a new version! For those not in the know: `skeletoken` is a set of Pydantic datamodels that fully describe the Hugging Face `tokenizer.json` format, so you can validate, edit, and transform tokenizers as typed objects instead of editing JSON files. So: version 0.4.0 is out, and it's a pretty big jump from 0.3.3, which is why I decided to write a blog post about it.

## Highlights

### Consolidating vocabularies

Adding a normalizer or pre-tokenizer can leave a vocabulary with entries that have become unreachable, or that now collide with another entry after the change. The new `consolidate_vocabulary` method finds and removes these automatically, while keeping the merge table and token IDs consistent.

```python
from skeletoken import TokenizerModel
from skeletoken.normalizers import ReplaceNormalizer

model = TokenizerModel.from_pretrained("bert-base-cased")
print("A" in model.vocabulary, "a" in model.vocabulary)
# True True

# Add a normalizer that maps "A" to "a", making the two tokens collide.
model = model.add_normalizer(ReplaceNormalizer(pattern="A", content="a"))
model = model.consolidate_vocabulary()

print("A" in model.vocabulary, "a" in model.vocabulary)
# False True
```

This is a generalization of `decase_vocabulary`, which [I wrote about previously](https://stephantul.github.io/blog/uncasing/): instead of manually removing uppercased tokens, this now works for any pretokenizer and/or normalizer. 

An interesting thing here is that, if tokens don't collide with others, they are kept. This allows your model to generalize to new tokens based on patterns you can define in your own pretokenizer:

```python
import re
from skeletoken import TokenizerModel
from skeletoken.normalizers import LowercaseNormalizer

model = TokenizerModel.from_pretrained("bert-base-cased")

tokenizer = model.to_tokenizer()
print(tokenizer.encode("Shakespeare shakespeare").tokens)
# Naively lowercasing would lose us `Shakespeare` as a token
# ['[CLS]', 'Shakespeare', 'shakes', '##pear', '##e', '[SEP]']

# Add a normalizer that lowercases
model = model.add_normalizer(LowercaseNormalizer())
model = model.consolidate_vocabulary()

tokenizer = model.to_tokenizer()
print(tokenizer.encode("Shakespeare shakespeare").tokens)
# Now we get to reuse the `shakespeare` token!
# ['[cls]', 'shakespeare', 'shakespeare', '[sep]']
```

### Safely resizing model embeddings

In general, editing a vocabulary changes token IDs. In most cases, this means your model's embedding table and your tokenizer silently drift apart. `skeletoken` now computes a `ModelDelta` (via `TokenizerModel.model_delta`) mapping old token IDs to new ones, and uses it to reshape a model's embedding table so tokens that survive an edit *keep their learned embeddings*.

```python
from transformers import AutoModel
from skeletoken import TokenizerModel
from skeletoken.external.transformers import reshape_embeddings

model_name = "bert-base-cased"
model = AutoModel.from_pretrained(model_name)
tokenizer_model = TokenizerModel.from_pretrained(model_name)

decased = tokenizer_model.decase_vocabulary()
new_model = reshape_embeddings(model, decased)

print(model.get_input_embeddings().weight.shape[0])       # original vocabulary_size
# 28996
print(new_model.get_input_embeddings().weight.shape[0])   # decased.vocabulary_size
# 25025
```

This is supported across `transformers`, `sentence-transformers`, `pylate`, and, new in this release, `model2vec` models, so you can prune or edit a vocabulary without retraining from scratch. While the `model2vec` integration is the only new one in 0.4.0, all four integrations were also reworked (#80) to be more consistent and correct.

## Bug fixes

* **Merging bug fixed** — a bug in how vocabulary merges were applied during edits is fixed, with a large batch of new integration tests across BERT, GPT-2, ModernBERT, multilingual-e5, and the `model2vec`/`sentence-transformers`/`pylate`/`transformers` integrations to guard against regressions (#81).
* **Preprocessor empty tokens** — the preprocessor now conditionally keeps empty tokens instead of always dropping (or always keeping) them, matching `tokenizers`' actual behavior (#83).

## Infrastructure

* CI now runs on Python 3.14.
* Releases are published via [trusted publishing](https://docs.pypi.org/trusted-publishers/) instead of a long-lived PyPI token.
