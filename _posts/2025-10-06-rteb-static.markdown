---
layout: post
title:  "Evaluating static models on RTEB"
date:   2025-10-06-00:00:00 +0000
categories: 
    - static models
header_image: /images/soup.jpg
math: true
---

The group of researchers associated with the [Massive Text Embedding Benchmark](https://huggingface.co/spaces/mteb/leaderboard) (MTEB) has released a new benchmark: the [Retrieval Text Embedding Benchmark](https://huggingface.co/blog/rteb). As you may know, MTEB ranks models on their ability to perform well at a variety of tasks in a zero-shot setting, and is meant to reflect how well your model transfers to new tasks. Ranking high on MTEB can make or break your model, so it has become something that people optimize for, and as [Goodhart put it](https://en.wikipedia.org/wiki/Goodhart%27s_law): "when a measure becomes a target, it ceases to become a good measure". 

The mechanism behind Goodhart's law is particularly problematic for MTEB, since all the datasets and evaluations behind it are completely open, making it feasible to hill-climb MTEB without actually training directly on any data from the benchmark. ([^1]) RTEB solves this issue by keeping a portion of the leaderboard private, a practice that used to be common in so-called shared tasks. ([^2]) Users wishing to appear on the leaderboard need to provide their model and have it tested on the private subset. This solves the issue of adversaries with a lot of compute being able to hill-climb the leaderboard by themselves. The downside of doing this is obviously that keeping the leaderboard up to date is a substantial effort. In addition, RTEB exclusively focuses on retrieval, and only uses datasets that are relevant for retrieval.

### Training static models

By and large, there are two good ways to train a static model: ([^3])

1. Knowledge distillation: used to create the [potion models](https://huggingface.co/collections/minishlab/potion-6721e0abd4ea41881417f062) by [Minish](https://minish.ai/). This approach performs basic knowledge distillation using a larger teacher model and the cosine similarity or MSE as a loss function. 
2. Supervised training: detailed in [Tom Aarsen's blog post](https://huggingface.co/blog/static-embeddings). This is simply performing supervised training using, e.g., a ranking loss, on very large datasets, without doing any finetuning.

As far as I could tell, both approaches are roughly competitive. Here are the scores for the models on MTEB, where `potion` models are trained via knowledge distillation, and the `static-.+mrl` models are trained on large datasets of sentences.

| Name  | MTEB avg score         | MTEB subset |
|------|-------------------------|---------|
| [potion-multilingual-128m](https://huggingface.co/minishlab/potion-multilingual-128M)   | 47.23                  | multilingual |
| [static-similarity-mrl-multilingual-v1](https://huggingface.co/sentence-transformers/static-similarity-mrl-multilingual-v1) | 47.24    | multilingual |
| [potion-base-8m](https://huggingface.co/minishlab/potion-base-8M)   |  53.3  |  english
| [static-retrieval-mrl-en-v1](https://huggingface.co/sentence-transformers/static-retrieval-mrl-en-v1)  | 51.25                |  english

As you can see, `potion-base-8m` is on average better than `static-retrieval-mrl-en-v1`. At retrieval, however, the supervised model is better than the `potion` model. For the multilingual models, knowledge distillation and the supervised approach seem to do equally well. The conclusion so far: training on sentence datasets leads to pretty good general models, but very good models at whatever you are training on (retrieval), and multilingual semantics can be learned really well from sentence datasets, even without prior language model training.

Now, let's turn to RTEB. As noted above, RTEB is specifically meant for retrieval, and also has an English and multilingual subset. This allows us to answer the following question: does knowledge distillation-based training lead to better performance on held-out data than straight supervision? Because we have English and multilingual models in both conditions, we have a very nice way to test. My personal prediction is that knowledge distillation would be better than supervision; even though the supervised models have been trained on large amounts of data, they have been trained to solve specific problems. Knowledge distillation, on the other hand, is about generally mimicking a larger model, and should thus generalize better to unseen datasets. 

### Results

Overall, supervised models outperform knowledge-distilled ones, particularly on the private leaderboard. I also didn't do anything; [Kenneth Enevoldsen](https://www.linkedin.com/in/kennethenevoldsen/) ran the models on RTEB. ([^4]) I can't really report much more than the actual results, so let's dive right in. Note that, as before, the top two rows are on the English subset, while the bottom ones are on the multilingual subset. 

| Name  | RTEB public score         | RTEB private score |
|------|-------------------------|---------|
| [potion-multilingual-128m](https://huggingface.co/minishlab/potion-multilingual-128M)   | 23.23                  | 36.63 |
| [static-similarity-mrl-multilingual-v1](https://huggingface.co/sentence-transformers/static-similarity-mrl-multilingual-v1) | 24.54    | 43.73 |
| [potion-base-8m](https://huggingface.co/minishlab/potion-base-8M)   |  24.11  |  37.45
| [static-retrieval-mrl-en-v1](https://huggingface.co/sentence-transformers/static-retrieval-mrl-en-v1)  | 29.09                |  44.48

As you can see both of the supervised mrl models surpass their knowledge distilled counterparts on the private set. This is especially striking for the multilingual model: `potion-base-128m` tracks the mrl model very closely on the public set, but is much worse on the private set. This is very interesting, and ran counter to my expectation, as all these models were more or less evenly matched on the full MTEB set.

### Discussion

This provides some interesting insights for future models. Knowledge distillation is basically free: all you need is a model for whatever domain you want, and a relatively small corpus, but it does not perform as well as supervised learning, even if the data you have does not match your task directly. The main data point here is the performance of `static-similarity-mrl-multilingual`, which was only trained on similarity datasets, and not on retrieval, but still outperforms the knowledge distilled `potion` model on retrieval.

Another interesting observation missing from this chart is hybrid models; it could be that first performing knowledge distillation and then supervised learning ([^5]) outperforms doing either of them alone. One issue with static models, however, is that they are extremely susceptible to catastrophic forgetting; without any intervening model, the embeddings just change shape to suit whatever task you train them on.

### Conclusion

I think that hybrid training and knowledge distillation, and especially knowledge distillation on a larger and more diverse set of documents, could be beneficial. In addition, I think the solution space of knowledge distillation for static models remains unexplored. For example, I don't think anyone has trained a model using hard negatives, or using logit scores. These things will surely be tried by someone ([^6])

### Footnotes

[^1]: This is obviously against the spirit of the leaderboard, but also how science progresses. This is not necessarily an issue, because when parties are forced to disclose whatever made them take a step up the hill, we learn a little bit. The main issue, in my opinion, is that a single user takes many steps in private, and then only discloses the tricks that worked.

[^2]: See for example the *SEM shared task series, which gave us the well-known sts datasets.

[^3]: Note that I am excluding [model2vec](https://github.com/MinishLab/model2vec) from training because I view that as an initialization strategy. Models that come straight from model2vec are not competitive; performing knowledge distillation or training is always better.

[^4]: Thanks! 🙏🙏🙏

[^5]: Or the other way around, or both

[^6]: Probably me...