# Level Up Your Search

You’ve implemented a first version of search. Now you want to improve it. A great search experience can make users happier and more engaged. Today, we’ll breakdown how to measure search, analyze the data, and deal with common issues.

## Measure

The first step to improving is measuring. This allows us to see where we stand and track progress over time.
First, think about what you want users to do. For an e-commerce site, you likely want users to add an item to their cart or make a purchase. You may want to have multiple conversion goals, but for now, let’s start with one.

When a user searches, track the:

- query
- user id (or visitor id)
- number of results
- time

When a user converts, track the:

- result id
- position
- time

We also want a flag for excluding searches from admins and bots from analysis. We can use the user agent to detect bots. There are many open source projects for this. It’s also good to exclude users who search excessively. They may be bots as well, and at the very least can throw off analysis. We can retroactively determine this and update the flag.

We want to collect all the data in a single table to simplify analysis. Our initial searches table will have:

- query
- user_id
- results_count
- searched_at (time)
- result_id
- position
- converted_at (time)
- exclude (boolean)

If you have different results for different stores or regions, store those values as well.

You could also have a separate table for conversions to record multiple conversions for a single search, in addition to storing the first conversion in the searches table. If users can sort or filter results, we recommend storing those actions in a separate table as well. However, these are outside the scope of this post.

Now, let’s analyze.

## Analyze

With the fields above, you can calculate:

- top queries
- overall conversion rate
- queries with a low conversion rate
- queries with no results (or few results)
- average searches per user
- average time to conversion
- average position of conversions

There are a number of things we can improve (time to conversion, position of conversions, etc), but it often makes sense to start with the overall conversion rate. Plot it over time so we can see our progress.

Start by getting the top 100 queries and sorting by conversion rate (lowest first). If we’ve set the `exclude` flag properly, the data should be pretty clean. For each query, perform a search and try to identify the issue. Likely, a number of themes will emerge.

## Common Issues

A few common issues are:

- missing results
- irrelevant results
- unavailable results

We’ll discuss each in detail.

Our general approach to improve will be to first improve *recall* (fix missing results), then improve ranking for better *precision* (fix irrelevant results).

## Missing Results

*Also known as poor recall*

There can be a number of reasons results aren’t returned when you expect them to be. We’ll discuss five common ones:

1. Stemming
2. Synonyms
3. Misspellings
4. Special characters
5. Spaces

### Stemming

If a user searches for `apples`, we want to return results with `apple`. Stemming is one way of accomplishing this. Stemming reduces a word to its stem (similar to its root word). There are [many different algorithms](https://en.wikipedia.org/wiki/Stemming) for stemming. [Porter](http://snowball.tartarus.org/algorithms/porter/stemmer.html) is a popular one for English. With the Porter stemmer, both `apples` and `apple` stem to `appl`.

### Synonyms

When a user searches for `coke`, they likely want results with `coca-cola` to be returned as well. The same goes for `tissues` and `kleenex`.

A key consideration in how to implement this is the difficulty of updating synonyms. With Elasticsearch, we recommend doing synonym expansion at query time so you can update synonyms without reindexing. You can read more about the tradeoff [here](https://www.elastic.co/guide/en/elasticsearch/guide/current/synonyms-expand-or-contract.html).

There are some common symbols you may want to expand like `&` to `and` and `%` to `percent`. Also, the [WordNet database](https://en.wikipedia.org/wiki/WordNet) has a list of English synonyms. However, loading the entire database can significantly impact performance, so we recommend building a smaller list by hand.

### Misspellings

We aren’t always great spellers. We type `zuchini` when we want `zucchini` and `siracha` when we want `sriracha`. Common misspellings will emerge and can be mapped to correct spellings. However, this won’t catch the long tail of typos.

A more general approach is fuzzy searching. This typically returns results that are within a certain [edit distance](https://en.wikipedia.org/wiki/Edit_distance). The Damerau–Levenshtein distance is a good choice. It counts an edit as an insertion, deletion, substitution, or transposition of two adjacent characters. For instance, `hello` and `helm` have an edit distance of two (substitute `l` for `m` and delete the `o`). Also, it’s available on popular search engines. An edit distance of one is a good place to start.

There are a few downsides to fuzzy searching to be aware of. The biggest is it can return irrelevant results. For instance, a search for `beet` will return `beef` and `beer`. It’s also less performant.

Both of these can be addressed by fuzzy searching selectively. If a search returns many results without fuzziness, fuzzy searching is unlikely to be useful. So for each search, first perform without fuzziness, and only if there are too few results, search again with it.

### Special Characters

Some results may have special characters, like `jalapeño`. We want a match if the user searches `jalapeno` (without the enye). ASCII folding is a technique to map characters to their ASCII equivalent. This maps  `ñ` to `n`. For English, this works well, but it can be problematic for other languages.

### Spaces

Search engines often use tokenization to split text into words, so whitespace (or lack of whitespace) can be problematic. Let’s examine the phrases `dish washer` and `dishwasher`. We likely want them to return similar results. We could map them as synonyms, but it would be tedious to do this with many phrases.

One approach is to use word n-grams, or shingles. With this approach words are combined together in tokens, so `red dish washer` is tokenized to `reddish` and `dishwasher`. When indexing, index the shingles in addition to the individual words. When querying, try both the original query as well as shingles. Let’s look at two examples.

#### Example 1: Spaces in Result

You have an item named `Red Dish Washer` and a user searches for `red dishwasher`. The tokens produced are:

- `red`, `dish`, `washer`, `reddish`, `dishwasher` for the item
- `red`, `dishwasher` for the original query
- `reddishwasher` for the query with shingles

All tokens in the original query match the item, so it’s a match.

#### Example 2: Spaces in Query

You have an item named `Red Dishwasher` and a user searches for `dish washer`. The tokens produced are:

- `red`, `dishwasher`, `reddishwasher` for the item
- `dish`, `washer` for the original query
- `dishwasher` for the query with shingles

All tokens in the query with shingles match the item, so it’s a match.

> Note: the query `red dish washer` will not match, as it’s tokenized to `reddish` and `dishwasher`. One way around this is to include the individual words as tokens as well and use an `OR` condition, but this can have other side effects.

## Irrelevant Results

*Also known as poor precision*

We know how to improve recall, so let’s focus on precision - specifically, the precision of our top-most results. We’re not too concerned if we have irrelevant results that appear at the bottom, although we have an approach to mitigate this for a specific case.

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

### Words Within Concepts

When searching for `butter`, you probably aren’t looking for `peanut butter`. An easy fix is to add a *NOT* condition for `peanut butter` to this search. Keep a mapping of these and apply them as needed.

## Unavailable Results

Sometimes, you understand exactly what the user wants, but it’s not available. You may have personally encountered this problem with Netflix. They know what movie you want to watch, but it’s unavailable for streaming. At Instacart, people sometimes search for products we don’t sell - like cigarettes - or produce that’s only available during certain seasons - like strawberries.

In this case, you can explain it’s unavailable and show related items.

## Other

You’ll likely find other issues that don’t fit into the categories above. You should still classify them. A few we’ve encountered at Instacart are:

### Unexpected Searches

People often type the name of a retailer, like Whole Foods or Costco, into the the search box when trying to change stores. We now automatically switch stores when this happens.

### Missing Images

People like to see images of the products they’re buying, so having search results with few images leads to low conversions. In this case, we need to get images. A few different ways we’ll do this are licensing the images through a 3rd party API, reaching out to manufacturers directly, or photographing them ourselves.

### Missing Products

In the early days, products sometimes weren’t added to our catalog, even though we carried them. Once, we were missing all the cream cheese from a popular retailer. This was easy to identify after looking at search data. The fix was pretty straightforward: add them.

## Conclusion

This should give you a nice foundation for improving search. If you use Ruby, [Searchkick](https://github.com/ankane/searchkick) and [Searchjoy](https://github.com/ankane/searchjoy) together provide much of this functionality out of the box.
