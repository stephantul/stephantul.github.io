---
layout: post
title:  "Static late interaction models"
date:   2025-09-30-00:00:00 +0000
categories: 
    static models
header_image: /images/El_Aquelarre.jpg
math: true
---

Late interaction is an interesting paradigm for computing the similarity between two documents, and can be seen as a hybrid of sparse and dense retrieval. In this post, I will show how [static models](https://huggingface.co/blog/static-embeddings) in a late interaction setting actually reduce to sparse models. I will also argue that, in absence of empirical evidence to the contrary, there's no good reason to assume that static late interaction models will be much better than their dense counterparts. But first, let's dive into some fundamentals: I'll explain what sparse retrieval and dense retrieval are, and how late interaction fits in with both of those paradigms.

### Sparse

Sparse retrieval assigns each individual token one or more coefficients, and only scores tokens when they are present in the document. For example, a query like "pet stores" will only return documents that contain those terms. In other words, sparse retrieval does not retrieve semantically related words; it only indexes documents based on the terms that are actually present. Examples of sparse retrieval techniques are [SPLADE](https://arxiv.org/abs/2107.05720), [DeepImpact](https://arxiv.org/abs/2104.12016), [BM25](https://en.wikipedia.org/wiki/Okapi_BM25) and [uniCOIL](https://arxiv.org/abs/2106.14807). Sparse retrieval tends to be precise because it matches terms exactly, but for the same reason also has trouble bridging the gap between semantically related terms. ([^1]) In general, sparse retrievers don't do well if there's little to no lexical overlap between queries and documents, or if there's lots of semantic ambiguity.

### Dense

In contrast, in dense retrieval, we assign a single vector to a whole document, and just compute dot-product based similarities between vectors to find the most similar ones. Putting everything in a single vector means that we mix up all words in the documents, thus allowing for queries with no lexical overlap to still retrieve relevant documents. The downside of this is that things can get too mixed up; a vector can implicitly model many related things, and it is difficult to predict a priori which vectors will match and why. In a way, it is naïve to expect a single vector to fully express the semantics of a document of arbitrary length.

### Late interaction

Which brings us to late interaction. In late interaction, we use a dense model ([^2]) to create one vector for *each token in the document*. If this sounds excessive, don't worry, there's many tricks to alleviate the burden of storing and retrieving this many documents. ([^3]) At query time, instead of calculating the dot product between vectors, we calculate the `maxsim` similarity. For a given query `Q` with `m` tokens and document `D` with `n` tokens, the similarity is as follows:

$$
s(Q,D)
= \sum_{i=1}^{m} \max_{1 \le j \le n} \;\Big\langle \widehat{\mathbf q}_i,\;\widehat{\mathbf d}_j \Big\rangle
$$

So, for each query token, we first calculate the similarity ([^4]) to each document token, and then take the max of those similarities. The sum over all of the query tokens is the `maxsim` score.

`maxsim` allows late interaction models to attach scores to specific tokens, like sparse retrieval models, but also allows for a graded similarity between related tokens, like dense models (and unlike sparse models). As such, we can think of late interaction models as a hybrid between dense and sparse models. There's many other aspects to dig into, which I won't cover here, so please read [one](https://medium.com/@varun030403/colbert-a-complete-guide-1552468335ae) [of](https://jina.ai/news/jina-colbert-v2-multilingual-late-interaction-retriever-for-embedding-and-reranking/) [the](https://weaviate.io/blog/late-interaction-overview) [many](https://qdrant.tech/articles/late-interaction-models/) [good](https://www.answer.ai/posts/colbert-pooling.html) posts on the subject.

### Static models

The reason late interaction models work is not just because of `maxsim`, but also because the underlying models are _trained_ to maximize the similarity between a query token and a related document token. These models are _contextualized_, which means that the model produces different vectors for tokens in different contexts. Static models, on the other hand, always produce the same vector for each token, regardless of the context. This makes static vectors worse, but also much faster. Why and when this is useful is the topic of an upcoming post, but for now let's assume this is a useful property.

### Static late interaction

Now, I will argue that `maxsim`, when applied to a static model, implicitly leads to a sparse model. First, recall that, in a static model, every occurrence of a token always gets the same vector. This also implies that the similarity between two tokens is always exactly the same: if `dog` and `cat` always get the same vector, then `sim(dog, cat)` is always the same value. So, this gives us a nice optimization: we can precompute all possible similarities. For a vocabulary `V` with `t` tokens, this leads to a `t x t`-sized matrix, which we call `W`. Note that `W` is very big! For a vocabulary size of 30k, this already is a 900 million parameter matrix. In practice we can easily make this matrix extremely sparse by pruning any items below a certain threshold. ([^5])

Now, given `W`, `maxsim` reduces to:

$$
s(Q,D)
= \sum_{i=1}^{m} \max_{1 \le j \le n} \;W_{Q_iD_j}
$$

This formulation means that we only need to store token indices and compute query indices to get the same result as we would have gotten when storing all vectors and computing vectors at query time. ([^6]) We also still need to store `W`, however. In addition, it is also unclear whether this is actually efficient.

Fortunately for us there's yet another shortcut: for each token in document `D`, we can index the _columns_ from `W`, and take the *max*. This leads to a single `V`-sized vector, which we call `Y`. `Y` contains the pre-computed max from the document to each possible token. This effectively precomputes the `max` for each possible token for each document. So, if we do this, the only thing we need to do at query time is index this vector using the query tokens, and take the sum. Because the vectors are pretty sparse, and the sparsity is controllable, this leads to a small memory footprint, and small query-time compute. Here's the equation:

$$
s(Q,Y)
= \sum_{i=1}^{m} Y_{Q_i}
$$

To repeat: during query time, the only thing we do is index. The index consists of a single matrix, with a number of rows equal to the number of documents, and columns equal to `t`, the vocabulary size. 

One question this raises is whether, for a decently-sized corpus, this matrix is actually smaller than `W`. Trivially, it might seem that if the number of documents approaches the vocabulary size, it starts becoming larger than `W`. But this ends up not being the case, because the `W` is much sparser than the document matrix; because a document vector is the max of a lot of tokens, it tends to have many more non-zero coefficients. So, in practice, if space is an issue, it might actually be better to still use `W`. If speed is a concern, it might be better to bite the bullet, and store the extra coefficients.

### Sparsity

The older people among you will point to this and say: this is just a sparse index, but with soft weights on related terms! ([^7]) And you would be right! In fact, if we set the similarity threshold on `W` to 1.0 we get a very bad version of BM25. ([^8]) Note that this behavior does not appear because we perform some magic trick or manipulation: it is inherent to the way `maxsim` works. So even if you compute `maxsim` as in the original equation, you will get this BM25-like behavior.

This explains why I think that just computing the `maxsim` with a static model as a regular late interaction model will never work well: BM25 contains a lot of cool tricks to make sure retrieval works well, including different weighting schemes for queries and documents, a length bias, and two tunable parameters. These are all missing from this algorithm. An interesting task, then, could be to re-add these terms: [model2vec](https://github.com/MinishLab/model2vec) for example, adds weighting by inflating and shrinking the norms by token frequency. These weighting terms can be re-added on the query tokens, or put in the index. Similarly ([^9]), the length bias in BM25 can also be integrated into this formulation. 

### Conclusion

This is all preliminary theoretical work, but which can be very promising. One thing that specifically is interesting is _asymmetric_ static models, i.e., using different static models to encode queries and documents, which is something I am actively working on. It is currently unclear whether training static models as late interaction models is actually useful. I have trained some static models using [PyLate](https://github.com/lightonai/pylate), but this did not lead to good results; training them as regular dense retrievers works much better. More research is needed, as always. Feel free to reach out if you have ideas, I'm always open to talk.

### Acknowledgments

* Thanks [jonah](https://x.com/drexalt) for proofreading and helpful suggestions about SPLADE.
* Thanks [Ben](https://x.com/bclavie) for suggesting blogs to link to.


[^1]: This is typically alleviated through query expansion techniques. SPLADE is also notable in that it automatically performs query/term expansion within the model, in addition to scoring terms that are present.

[^2]: These dense models are specifically trained to be late interaction models, but their cores are just pre-trained transformers, like the ones we use for dense retrieval. For training details, see [the colbert paper](https://arxiv.org/abs/2004.12832) and [the colbertv2 paper](https://arxiv.org/abs/2112.01488). You can use [PyLate](https://github.com/lightonai/pylate) to train, it's easy!

[^3]: Examples of this include [MuVERA](https://arxiv.org/abs/2405.19504), [FastPLAID](https://www.lighton.ai/lighton-blogs/fastplaid), [maxsim-cpu](https://www.mixedbread.com/blog/maxsim-cpu) and probably many others.

[^4]: In the equation we use the dot product similarity, but the cosine similarity can also be used.

[^5]: This pruning is empirically justified: because of the maxsim, it is unlikely that tokens with very low similarities ever get selected. And even if they do get selected, it is unlikely that this will lead to a meaningful difference in selected documents. The proof of the pudding is in the eating, however.

[^6]: In practice, ~90% of time is spent on tokenization, so this isn’t the big win it seems.

[^7]: If you are young and noticed this: good job buddy! 

[^8]: Bad because it does not have any of the things, i.e., length, query weights, IDF, that make BM25 actually good.

[^9]: haha