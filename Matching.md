# Matching

The goal of matching is to return relevant results without returning irrelevant ones. There are a number of general techniques you can use to make this happen.

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

### False Matches

When searching for `butter`, you probably aren’t looking for `peanut butter`. An easy fix is to add a *NOT* condition for `peanut butter` to this search. Keep a mapping of these and apply them as needed.

### Unavailable Results

Sometimes, you understand exactly what the user wants, but it’s not available. You may have personally encountered this problem with Netflix. They know what movie you want to watch, but it’s unavailable for streaming. At Instacart, people sometimes search for products we don’t sell - like cigarettes - or produce that’s only available during certain seasons - like strawberries.

In this case, you can explain it’s unavailable and show related items.
