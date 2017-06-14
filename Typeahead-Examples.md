# Typehead.js Examples

Typeahead.js offers prefetch, which loads terms in a single request after the initial page load.  This keeps the initial page load fast and results show up instantly as the user types. However, prefetch uses local storage and [it’s not recommended to be used for the entire data set](https://github.com/twitter/typeahead.js/blob/master/doc/bloodhound.md#prefetch), so we use a custom prefetch that doesn’t use local storage.

```js
engine = new Bloodhound({
  datumTokenizer: Bloodhound.tokenizers.obj.whitespace('value'),
  queryTokenizer: Bloodhound.tokenizers.whitespace,
  limit: 4,
  local: []
})
engine.initialize()

// fetch suggestions
$.getJSON('/suggestions', function (suggestions) {
  engine.add(suggestions)
})

$searchInput.typeahead({highlight: true}, {source: engine.ttAdapter()})
```

Measure typed query (only populated when autocompleted)

```js
$searchInput.on('keyup', function () {
  typedQuery = $searchInput.typeahead('val')
}).on('typeahead:selected typeahead:autocompleted', function () {
  $('#typed_query').val(typedQuery) // autocompleted!!
})
```

Measure typing time

```js
$searchInput.on('keyup', function (e) {
  if (!typingStartedAt && e.keyCode != 13) {
    typingStartedAt = new Date()
  }
})
$searchInput.closest('form').on('submit', function () {
  if (typingStartedAt) {
    $('#typing_time', ((new Date()) - typingStartedAt) / 1000.0)
    typingStartedAt = null
  }
})
```
