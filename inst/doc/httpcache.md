<!--
%\VignetteEngine{knitr::knitr}
%\VignetteIndexEntry{Improving API client performance with httpcache}
-->

# Improving API client performance with `httpcache`

`httpcache` provides the
HTTP verb functions `GET`, `PUT`, `PATCH`, `POST`, and `DELETE`, which are drop-in
replacements for those in the [httr](https://github.com/hadley/httr) package. So, to take advantage of these cache-aware functions, all you need to do is load `httpcache` instead of `httr`, or in package development, import from `httpcache`.


```r
library(httpcache)
```





```r
system.time(a <- GET("http://httpbin.org/get"))
```

```
##    user  system elapsed 
##   0.013   0.003   0.599
```

```r
system.time(b <- GET("http://httpbin.org/get"))
```

```
##    user  system elapsed 
##       0       0       0
```

Notice how the second request returns instantly. It's reading from cache--there's no communication with a remote server, so no network latency and no server processing time. Remember: the fastest API request is the one you don't have to make.

Reading from the query cache yields exactly the same response as if we had contacted the server. We can confirm:


```r
identical(a, b)
```

```
## [1] TRUE
```

## Query logging

How do we know, other than by the faster response, that we're hitting cache and not making server requests?

When designing API clients in R, logging is an invaluable tool for understanding and improving request patterns. As you build layers of abstraction on top of the direct HTTP requests, it can be easy to make inefficient or repetitive requests that degrade performance for your users and impose unnecessary load on your servers. You can't improve what you can't measure, and the logging tools included in `httpcache` can help you measure.

Let's clear the cache and repeat that exercise, this time with logging enabled. Use `startLog` to enable the request log. `startLog` takes a file or connection argument, which it passes to `cat` for log writing. The default, same as for `cat`, prints to the standard output--your display, in an interactive session. (See `?cat` for details.)


```r
clearCache()
startLog()
a <- GET("http://httpbin.org/get")
```

```
## 2016-02-26T23:43:16 HTTP GET http://httpbin.org/get 200 0.02644 
## 2016-02-26T23:43:16 CACHE SET http://httpbin.org/get
```

```r
b <- GET("http://httpbin.org/get")
```

```
## 2016-02-26T23:43:16 CACHE HIT http://httpbin.org/get
```

Notice how the first request results in an "HTTP GET" and a "CACHE SET", while the second one gets a "CACHE HIT" and does not touch "HTTP". From this log output, we can conclude that the query cache is working.

### Log analysis

You can also pass a file name to `startLog`. This makes it easier to read the log output back in as a `data.frame` and analyze it quantitatively. We'll do an example of that below.

### Custom logging

You may want to send other events to the log, interspersed with your HTTP requests, whether for their own sake or to see how work done in R outside of the HTTP layer maps onto your server traffic. The function `logMessage` writes to the connection specified by `startLog`, and it is available for general use. For error logging, the `halt` function wraps `stop` and sends a message to the log (it also makes the awkwardly named `call.` argument to `stop` default to `FALSE` for cleaner error messaging).

## Cache invalidation

As the [saying (or joke, depending on the version)](http://martinfowler.com/bliki/TwoHardThings.md), cache invalidation is one of the two hard problems in computer science. The trouble with caching what the server serves is that the server is the source of truth, and if the state of data on the server changes, our local copy of the data is stale. In some applications and with some APIs, we have no idea when the server state changes, but in many cases, the source of change on the server is actions that we initiate ourselves. In these cases, a local query cache is more feasible, and cache invalidation more tractable.

`httpcache` provides some functions to direct cache invalidation. We've seen one already, `clearCache`, which wipes the entire cache. Other functions give more surgical control. `dropOnly`
invalidates cache only for the specified URL. `dropPattern` uses
regular expression matching to invalidate cache. `dropCache` is a
convenience wrapper around `dropPattern` that invalidates cache for
any resources that start with the given URL.

Depending on the API with which you're communicating, you may not need to use those cache-invalidation functions directly, or you may need them only infrequently. `httpcache` was designed with [RESTful](https://en.wikipedia.org/wiki/Representational_state_transfer) APIs in mind, particularly those that expose resources that contain collections of entities that can be created, replaced, updated, and deleted with POST, PUT, PATCH, and DELETE, respectively. Consequently, these four HTTP verb functions are built with default cache invalidation actions: `POST` invalidates cache only for the request URL (`dropOnly`), for the case where POST creates a new entity appearing as a subresource. For example, if `GET http://api.example/projects/` returns a catalog of project entities, and POST to `http://api.example/projects/` creates a resource at `http://api.example/projects/new_id/`, we need to bust cache for the project catalog on POST, but our cached responses for resources such as  `http://api.example/projects/old_id/` should still be valid. The other three functions, `PUT`, `PATCH`, and `DELETE`, by default drop cache for the request URL and everything "below" it (`dropCache`). Continuing with the example, if we modify `http://api.example/projects/new_id/`, we should invalidate cache for that resource and for other resources appearing as subresources of it, such as `http://api.example/projects/new_id/users/`.

These verb functions (`POST`, `PUT`, `PATCH`, and `DELETE`) all take a `drop` argument, which defaults as described above. To override them, you can specify a different call other than `dropCache(url)` or `dropOnly(url)`, or you can pass `drop=NULL` and call the cache-invalidation functions directly. Depending on your API and your usage of it, however, `httpcache`'s cache management may just work for you with no additional effort.

## Example

For a more complete example of the caching, logging, and invalidation features, we'll use a mock HTTP interface that allows demonstration of the features of `httpcache` without requiring a network connection. Refer to the [setup code in the test suite](http://github.com/nealrichardson/httpcache/blob/master/tests/testthat/helper-mocks.R) for specifics. We'll make a series of requests, both reads and writes, and we'll capture the action with the logging tools.
<!-- do requests. look at log. show cache invalidation. then show reading without internet -->
