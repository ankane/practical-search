# Autocomplete

Autocomplete can save users time and help them find what they’re looking for. A common approach for e-commerce apps is to use popular queries. We can:

- start with common queries
- filter duplicates, misspellings, and queries without results
- approve suggestions

Then, we can implement and measure.

### Top Queries

First, we need to measure search. You can read more about how to do this [here](Level-Up-Your-Search.md).

Once it’s measured, get a list of queries and the number of distinct users who searched for it. With SQL:

```sql
SELECT LOWER(query), COUNT(distinct user_id) FROM searches
WHERE exclude = FALSE
GROUP BY LOWER(query)
HAVING COUNT(distinct user_id) > 5
```

> Note: If you have different results for different stores or regions, do this seperately for each of them. For instance, `Kirkland Signature` may be a top query in one store (Costco) but not in other stores.

### Duplicates

The first thing you’ll notice is many queries are similar.

- apple, apples (plural)
- hand soap, handsoap (space)
- ben & jerry's, ben and jerry's (ampersand)
- organic milk vs milk organic (order)
- amy's, amys (apostrophe)

We need to:

1. detect these
2. decide which to show

We can use a custom hashing algorithm for detection.

1. tokenize (whitespace works)
2. stem each token
3. sort
4. concat (without spaces)

For instance, `organic soaps` and `soap organic` both hash to `organsoap`.

When deciding which to show, a good heuristic is to show the most searched. However, you may like `amy's` to show up over `amys`. In this case, you may want to check against brands and prefer them.

### Misspellings

We don’t want `zuchini`, `zuchinni`, and `zucchini` showing up in your suggestions. We could use a spelling library like [Hunspell](http://hunspell.sourceforge.net/) or [Aspell](http://aspell.net/). However, most of the time, queries will be domain-specific.

One solution is to create your own corpus. At Instacart, we use product names. Since even these could have misspellings, set a minimum number of times a word appears before being added to the corpus.

### Ensure Results

We don’t want to suggest a query with no results, so each suggested query should be checked for results. If you use fuzzy searching, turn it off for this.

### Approve Suggestions

Once the automated work has been completed, you should look over the suggestions and bulk approve them by hand. If you have multiple stores or region, you can take shortcuts here: if a query is already approved for one store, approve it for others.

## Implement

Once we have suggestions, it’s time to implement and measure. There a number of decisions to make:

- which library to use
- the number of suggestions to show
- how to rank

We’ll take it one by one.

### Libraries

Here’s our list of recommended client libraries:

- web - [Typeahead.js](https://twitter.github.io/typeahead.js/)
- iOS - [MLPAutoCompleteTextField](https://github.com/EddyBorja/MLPAutoCompleteTextField)
- Android - [AutoCompleteTextView](http://developer.android.com/reference/android/widget/AutoCompleteTextView.html)

### Number of Suggestions

The number of suggestions varies from site-to-site.

Site | Suggestions
--- | ---
Amazon | 8
Overstock | 10
Esty | 11
eBay | 12

Between 8 and 12 is probably good. You can always A/B test if needed.

### Ranking

To start, it’s easiest to rank by popularity (the distinct number of users who searched). Eventually, you could optimize for other objectives, like basket size.

### Tip

There’s no need to wait for a user to start typing to show suggestions. You can show popular ones as soon as the search box is focused.

## Performance

Responsiveness is essential for autocomplete. You should keep the number of network requests to a minimum and filter client-side when possible. If you have under 10k suggestions, prefech all of them in a single request.

## Measure and Analyze

We recommend adding a single field to the searches table to help with analysis: `typed_query`. It should be null for searches that weren’t autocompleted.

From this, you can analyze the percent of searches that use autocomplete. You can also see if the overall conversion rate increases after adding it (or do an A/B test).

## Conclusion

This should give you a nice foundation for getting started with autocomplete. Check out [Autosuggest](https://github.com/ankane/autosuggest) for a Ruby implementation.

If you use Typehead.js, we also have [examples](Typeahead-Examples.md) for how to prefetch and measure.
