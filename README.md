# Search Guide

Best search practices for developers and code to implement them. Let’s make search a better experience for our users.

This is a work in progress, built for the open-source community.  If you have great practices, articles, or videos, [please share](https://github.com/ankane/search_guide/issues/new).

:warning: Everything below is scattered and probably makes no sense

## Outline

- [Instrument](#instrument)
- [Analyze](#analyze)
- [Scoring Theory (Ranking)](#scoring-theory-ranking)
- [Semantic Search](#semantic-search)
- [Autocomplete](#autocomplete)

## Instrument

Track searches and conversions

Searches

- query (case-sensitive)
- user_id or visitor_id
- searched_at
- platform (`Web`, `iOS`, `Android`, etc)
- results
  - result_id
  - result_type
  - score
- facets
- typing_time
- response_time (for performance)
- typed_query (for autocomplete)
- exclude (boolean - `true` if admin or bot)
- referrer - which page was searched from
- user_agent
- ip
- other properties (like city, department, etc)

Events (to define conversions)

- search_id
- name (`Viewed`, `Added to cart`, etc)
- result_id
- position (in results)
- time

## Analyze

### Top Queries

Quick: top n ordered by lowest conversion rate asc
Better: use Wilson score for lowest max conversion rate, as it balances volume and conversion rate

Get all combinations of searches and conversions and bulk update queries

### Conversions

First, choose how to define conversions. [todo: explain more]

Look at:

- conversion rate
- time to result
- average position

### Sessions

todo: how to define a session (and possibly pseudo code)

- average number of searches per session

### No Results and Low Conversions

- what do people search for next
- manually review to try to find trends

### Autocomplete

Do autocompleted queries perform better?

Use linear classifier with features

### Performance

Search must be fast.

http://www.fastcompany.com/1825005/how-one-second-could-cost-amazon-16-billion-sales

[todo: how to improve performance. caching, etc]

## Scoring Theory (Ranking)

### Practical Sorting

default

http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/scoring-theory.html

### Conversions

Let your users order your results.

If a user searches for “ice cream” and adds Ben & Jerry’s Chunky Monkey to the cart (our conversion metric at Instacart), that result gets a little more weight for similar searches.

This works well when results are relatively static

Pros:

- works well without domain specific knowledge

Cons:

- bad at first
- how to handle new items?

### Machine Learning

[todo: which features influence the result, test algorithm against previous data]

## Analyze

- sessions
- queries
- strings of searches
- related searches

### Sessions

New session after conversion or amount of time passes

### Categorize Queries

[todo]

## Semantic Search

`organic` and `gluten free`

1. Detect
2. Take action

### Concepts

Butter vs peanut butter, peaches vs peach yogurt

### Synonyms

[todo]

### No Results

We know what you mean, but we don’t have it (Netflix movies, seasonal, alcohol)

1. Detect
2. Take action (show other items w/ the word peach, show items like peaches, show notice)

## Autocomplete

[Autocomplete](http://en.wikipedia.org/wiki/Autocomplete) makes users happy.

One approach is to use the most common queries.  Here’s an overview of the approach:

- start with common queries
- filter duplicates, misspellings, and queries without results
- approve suggestions

Once we have suggestions, we:

- implement
- instrument

This approach can be taken with any language.  The examples are mostly in Ruby.

Here’s our list of recommended client libraries:

- web - [Typeahead.js](https://twitter.github.io/typeahead.js/)
- iOS - [MLPAutoCompleteTextField](https://github.com/EddyBorja/MLPAutoCompleteTextField)
- Android - [AutoCompleteTextView](http://developer.android.com/reference/android/widget/AutoCompleteTextView.html)

### Top Queries

First, we need to track each query.  The key attributes for autocomplete are:

- query
- user_id (or visitor_id)
- exclude (admin or bot)

We want to filter queries from admins and bots and only count queries from distinct users for robustness.  We convert queries to lowercase since our search is case-insensitive, and we don’t need to consider queries under two characters.

The SQL looks something like:

```sql
SELECT
  DISTINCT COUNT(DISTINCT "searches"."user_id") AS count_user_id,
  LOWER(query) AS lower_query
FROM
  "searches"
WHERE
  "searches"."exclude" = 'f' AND LENGTH(query) > 2
GROUP BY
  LOWER(query)
HAVING
  COUNT(DISTINCT user_id) >= 5
ORDER BY
  count_user_id DESC, LOWER(query)
LIMIT
  10000
```

This finds the top 10,000 queries searched by at least 5 distinct users.

Queries from one store (or location) may be different than queries from another.  `Kirkland Signature` may be a top term in one store (Costco) but not in other stores, so you should generate a different set of suggestions for each store.

### Duplicates

The first thing you’ll notice in the top queries is that many are similar.

- apple, apples (plural)
- hand soap, handsoap (space)
- ben & jerry's, ben and jerry's (ampersand)
- organic milk vs milk organic (order)
- amy's, amys (apostrophe)

We need to:

1. detect these
2. decide which to show

A hashing algorithm is used for detection.

1. tokenize (whitespace works)
2. stem each token
3. sort
4. concat

In Ruby, it looks something like:

```ruby
tokenize(query).map{|q| stem(q) }.sort.join
```

This can lead to false positives - `straws` vs `straus` (the brand) - so you’ll want the ability to explicitly specify queries that are not duplicates.

By default, the most searched is shown. However, you may like `amy's` to show up over `amys`.  You can add preferred terms - brands in this case.

### Misspellings

You don’t want `zuchini`, `zuchinni`, and `zucchini` showing up in your suggestions.

The first thing to come to mind is use a spelling library, like [Hunspell](http://hunspell.sourceforge.net/) or [Aspell](http://aspell.net/).  However, words like `kombucha` is not included in the standard libraries.

One solution is to create your own corpus.  We use product names.  Since these could have misspellings, set a minimum number of times for each term.

### Ensure Results

We don’t want to suggest a query with no results, so each suggested query should be checked for results. If you use fuzzy searching (misspellings) in your search, turn it off for this.

```ruby
Product.search(query, misspellings: false).total_entries
```

### Approve Suggestions

Once the automated work has been completed, you should look over the suggestions and bulk approve them by hand. You can take shortcuts here as well: if a query is already approved for one store, approve it for other stores.

### Performance

#### Prefetch

Typeahead.js offers prefetch, which loads terms in a single request after the initial page load.  This keeps the initial page load fast and results show up instantly as the user types.  However, prefetch uses local storage and [it’s not recommended to be used for the entire data set](https://github.com/twitter/typeahead.js/blob/master/doc/bloodhound.md#prefetch), so we use a custom prefetch that doesn’t use local storage.

#### Caching

For the fastest response times, cache suggestions in an in-memory data store like [Memcached](http://memcached.org/).

[Action caching](https://github.com/rails/actionpack-action_caching) in Rails makes this simple.

### Number of Suggestions

The number of suggestions varies from site-to-site.

- Google: 4
- Bing: 8
- Amazon: 10+

We prefer to keep it short like Google.

### Design

Jamie Appleseed has an [excellent guide on design patterns](http://baymard.com/blog/autocomplete-design).

### Rank

Suggestions are ranked by popularity by default, but you could rank by other factors or customize by user.  More on this another time.

### Instrumentation

Add a `typed_query` attribute to set when a suggestion is chosen.  From this, you can compute metrics like:

- % of queries autocompleted
- # of characters saved

More importantly, good suggestions should reduce the time a user spends typing.

### Implementation

#### Model

- query
- score
- results_count
- duplicate, misspelling ???
- other properties

#### View

```js
engine = new Bloodhound({
  datumTokenizer: Bloodhound.tokenizers.obj.whitespace('value'),
  queryTokenizer: Bloodhound.tokenizers.whitespace,
  limit: 4,
  local: []
});
engine.initialize();

// fetch suggestions
$.getJSON('/suggestions', function (suggestions) {
  engine.add(suggestions);
});

$searchInput.typeahead({highlight: true}, {source: engine.ttAdapter()});
```

#### Instrumentation

Time to search for all queries

```js
$searchInput.on('keyup', function (e) {
  if (!typingStartedAt && e.keyCode != 13) {
    typingStartedAt = new Date();
  }
});
$searchForm.on('submit', function () {
  if (typingStartedAt) {
    $('#typing_time', ((new Date()) - typingStartedAt) / 1000.0);
    typingStartedAt = null;
  }
});
```

Typed query for autocompleted queries

```js
$searchInput.on('keyup', function () {
  typedQuery = $searchInput.typeahead('val');
}).on('typeahead:selected typeahead:autocompleted', function () {
  $('#typed_query').val(typedQuery); // autocompleted!!
});
```

## Contribute

This is a work in progress, built for the open-source community.  If you have great practices, articles, or videos, [please share](https://github.com/ankane/search_guide/issues/new).
