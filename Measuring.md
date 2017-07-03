# Measuring

The first step to improving is measuring. This allows us to see where we stand and track progress over time.

We want to know what users are searching for. For instance, what are the most popular queries. We also want to know if a search is successful. Did the user find what they were looking for? For an e-commerce site, a good indicator may be if she added item to her cart or made a purchase. You may want to have multiple conversion goals, but for now, let’s start with one.

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

Our general approach to improve will be to first improve matching, then improve ranking.
