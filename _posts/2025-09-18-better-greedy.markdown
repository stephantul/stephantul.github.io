---
layout: post
title:  "Better Greedy Tokenizers: Handling WordPiece's [UNK] Problem"
date:   2025-09-18-00:00:00 +0000
categories: tokenization 
---

In [a previous post](https://stephantul.github.io/blog/greedy/), I showed that making a tokenizer greedy, that is, always picking the longest matching subword like WordPiece does, can improve results without retraining. But `WordPiece` can unfortunately silently break your tokenization.

Consider this example:

```python
from tokenizers import Tokenizer

tokenizer = Tokenizer.from_pretrained("bert-base-uncased")
result = tokenizer.encode("talk" * 10_000).tokens
print(result[:10])
# ['[CLS]', '[UNK]', '[SEP]']
```

Instead of producing many repetitions of talk (or something else), the tokenizer outputs a single `[UNK]`.

This happens because WordPiece enforces a hard limit on the length of each run (the contiguous string passed to it after pretokenization). The length is determined by a parameter `max_input_chars_per_word`, which is set to 100 by default. As the name suggests, this parameter puts a maximum on the number of characters any pretoken going into the model has. Once you go over this limit, the model doesn't crash, but silently produces an `[UNK]` token.

What this means is that if you are used to BPE, `WordPiece` could often get you `[UNK]`. In practice, this makes it difficult to use `WordPiece` with highly multilingual collections, because it becomes much more probable to get long runs. In addition, it also makes it impossible to create a multi-word `WordPiece` tokenizer.

# Why does this parameter exist?

The reason why this happens is because of `WordPiece` itself. The `WordPiece` algorithm, as implemented in the Hugging Face tokenizers package is as follows:

For a given input string and a vocabulary of subwords, do the following:
1. Initialize two pointers, one at the start of the string, S, and one at the end of the string, E
2. Decrement E by 1 and see if the run from S to E forms a valid token.
3. Once you find a valid token, increment S by the length of the token you found.

If you ever don't find a token, you just emit `[UNK]` for the whole run. As you can probably see, this algorithm is quadratic as a function of input length. For every subword you find, you will skip to the end of the run and walk back to the near-start. There's lots of low-hanging fruit to make this more efficient, but that is not what this post is about. 

So, this also hopefully makes clear why `max_input_chars_per_word` exists: when encountering a single run of, say, 100k characters, the `WordPiece` inference algorithm could conceivably take hours. For example, on my machine, encoding a 500 character string takes 3.66ms (0.007 ms per character), a 5000 character string takes 638ms (0.12 ms per character, a 17x increase), while encoding a 50000 character string takes ... too long([^1]). It would be really silly to wait for such a long time.([^2])

# Fixing the parameter

Since we are stuck with the parameter, we might as well make the best of it.
As it turns out, there is a very nice solution we can leverage within the Hugging Face ecosystem: the `FixedLength` pretokenizer. The `FixedLength` pretokenizer simply splits strings up into tokens of a pre-specified length.

So picture this: you have a tokenizer you like, with a pretokenizer you like. But sometimes, due to the domain you find yourself operating on, you end up with a run that is longer than `max_input_chars_per_word`. Adding a `FixedLength` pretokenizer to the pretokenizer you already had solves exactly this issue: pretokenization proceeds as it normally would, but any runs coming out of your previous pretokenizer that are too long are then split up into usable chunks. Problem solved. The only issue you could run into is that you miss tokens you otherwise could have found.

# Implementation in skeletoken

This is fully implemented in [`skeletoken`](https://github.com/stephantul/skeletoken). Let's return to the example from the top of the article:

```python
from skeletoken import TokenizerModel

model = TokenizerModel.from_pretrained("bert-base-uncased")
model = model.make_model_greedy()

# Make fixedlength really low for demonstration purposes
model.pre_tokenizer.pretokenizers[-1].length = 10
tokenizer = model.to_tokenizer()

result = tokenizer.encode("talk" * 10_000).tokens
print(result[:10])
# ['[CLS]', 'talk', '##talk', '##ta', 'l', '##kt', '##al', '##kt', '##al', '##k']
```

As you can see, the third `talk` is chopped up into `##ta` and `lk`, because that's where the pretokenizer boundaries fell. In practice though, this should almost never occur or matter.

Note that this also makes it possible to use greedy tokenizers _without_ any form of pretokenization, and enables the use of multi-word units in `WordPiece` tokenizers: because multiword tokenizers generally don't pretokenize at all, any sequence over 100 characters would produce an `[UNK]`.

# Future work

In a future post I'll dive into how you can make a much faster greedy tokenizer by imposing specific restrictions on the tokenizer model, and then using the [Aho-Corasick algorithm](https://en.wikipedia.org/wiki/Aho%E2%80%93Corasick_algorithm) with backtracking to find subwords with much lower complexity.

[^1]: This actually took to long to run. Sorry!

[^2]: It is equally silly to not implement a more efficient variant when such things exist. For example, moving the end pointer not to the end of the string, but forward by the maximum subword length would completely solve this issue and literally lead to the same solution.








