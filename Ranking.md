# Ranking

There are many different strategies for ranking. If we want the most relevant results to show up first, we can take into account:

- number of terms that match
- significance of each term
- popularity of each result (such as number of times ordered or viewed)

One extremely effective strategy is the *popularity given the specific query*. To accomplish this, we’ll use the data we collected above.

### Conversions

Conversions are a great source of data for relevance. Algorithms like TF-IDF and BM25 work great when dealing only with text, but we now have powerful metadata.

If a user searches for `ice cream` and adds Ben & Jerry’s Chunky Monkey to the cart (our conversion metric at Instacart), that item should get a little more weight for similar queries. To prevent specific users from throwing off this approach, we count only one conversion per user.

The most basic method has two drawbacks:

1. New items are ranked last
2. Top result stay top results because top results convert better, even if they aren’t most relevant

There are number of ways to address each of these issues. We’ve opted for simple ones.

- For #1, assign new items a weight until enough data is collected.
- For #2, randomly penalize top results to give other results a better chance to convert.

Another good strategy for ranking, which can be combined, is personalization. However, let’s save that for another post. For now, we have a strategy for helping with precision when there’s little data and getting rid of the pesky irrelevant results at the bottom.

### Learning to Rank

Another more advanced strategy is called “learning to rank”. This uses machine learning to rank results. This typically has two steps:

1. Retrieve relevant results from your search engine
2. Rerank results with a machine learning model

The features of the model are often specific to the user searching, like whether they’ve bought the brand of the result before.

## Other

You’ll likely find other issues that don’t fit into the categories above. You should still classify them. A few we’ve encountered at Instacart are:

### Unexpected Searches

People often type the name of a retailer, like Whole Foods or Costco, into the the search box when trying to change stores. We now automatically switch stores when this happens.

### Missing Images

People like to see images of the products they’re buying, so having search results with few images leads to low conversions. In this case, we need to get images. A few different ways we’ll do this are licensing the images through a 3rd party API, reaching out to manufacturers directly, or photographing them ourselves.

### Missing Products

In the early days, products sometimes weren’t added to our catalog, even though we carried them. Once, we were missing all the cream cheese from a popular retailer. This was easy to identify after looking at search data. The fix was pretty straightforward: add them.
