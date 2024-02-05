---
layout: post
title:  "Fauxpen source software"
date:   2024-02-05-00:00:00 +0000
categories: culture
---

Recently, I've noticed a concerning trend in open source software, especially relating to machine learning and AI. When I started doing computational linguistics/NLP in 2015, there was very little in the way of open source machine learning software. The undisputed king of the hill, back then, was [scikit-learn](https://scikit-learn.org). To me, scikit-learn is still _the_ way to run an open-source machine learning software package: get a bunch of smart people to define a good API, which is iteratively refined over time with input from the community. Other really nice projects which I used often were [gensim](https://radimrehurek.com/gensim/) and [pattern](https://github.com/clips/pattern).

With the new influx of capital in the machine learning world, however, there seems to be a push towards a different kind of open source project. Instead of being run by small communities of passionate individuals, these projects are run by well-funded companies that often have multiple millions of dollars of (VC) funding. This money is spent to hire teams of talented software engineers to work on the project full-time, and spent to hire social media figureheads as community managers whose goal is to get as many active users as possible. The goal for these companies seems to be quite clear: to gain traction, or active users, which the company can later monetize using other projects, or, alternatively, to lock a significant part of the user base into their open source project, and later add features that can be monetized. In this way, these open source projects remind of the freemium app model which was popular on smartphones in the early 2010s: try the app for a little bit, pay for the other features. 

It is completely unclear to me what expectation these VCs have in investing millions in small teams with open source projects. I'm assuming that the investors assume that open source projects can überify their way into the market: capture significant market share by undercutting the competition in cost, and then raising prices. But I don't think open source software works this way. In fact, I think users that use an open source project for free are extremely difficult to convert to paid users, and doubly so if your users are actually software engineers that can _take your codebase and run it_. 

Some caveats before people get angry: 
* I don't think that making money from open source, or founding a company based on open source software, is a bad thing. I really admire what the [polars team](https://pola.rs/) is doing to improve dataframe support. 
* 

To celebrate my negative feelings towards these companies, I have come up with a new name for them: fauxpen source software projects (a portmanteau of _open_ and _faux_). While they add useful features to the ML ecosystem, they are not interested in working or interoperating with their competitors, or interoperating at all.
