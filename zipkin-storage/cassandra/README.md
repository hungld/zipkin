# storage-cassandra

This CQL-based Cassandra 2.2+ storage component includes a `GuavaSpanStore` and `GuavaSpanConsumer`.
`GuavaSpanStore.getDependencies()` returns pre-aggregated dependency links (ex via [zipkin-dependencies-spark](https://github.com/openzipkin/zipkin-dependencies-spark)).

The implementation uses the [Datastax Java Driver 3.x](https://github.com/datastax/java-driver).

`zipkin.storage.cassandra.CassandraStorage.Builder` includes defaults that will
operate against a local Cassandra installation.

## Logging
Queries are logged to the category "com.datastax.driver.core.QueryLogger" when debug or trace is
enabled via SLF4J. Trace level includes bound values.

See [Logging Query Latencies](http://docs.datastax.com/en/developer/java-driver/3.0/supplemental/manual/logging/#logging-query-latencies) for more details.

## Testing
This module conditionally runs integration tests against a local Cassandra instance.

Tests are configured to automatically access Cassandra started with its defaults.
To ensure tests execute, download a Cassandra 2.2-3.4 distribution, extract it, and run `bin/cassandra`. 

If you run tests via Maven or otherwise when Cassandra is not running,
you'll notice tests are silently skipped.
```
Results :

Tests run: 62, Failures: 0, Errors: 0, Skipped: 48
```

This behaviour is intentional: We don't want to burden developers with
installing and running all storage options to test unrelated change.
That said, all integration tests run on pull request via Travis.

## Tuning
This component is tuned to help reduce the size of indexes needed to perform query operations. The most important aspects are described below. See [CassandraStorage](src/main/java/zipkin/storage/cassandra/CassandraStorage.java) for details.

### Service and span name indexing
Redundant requests to store service or span names are ignored for an hour to reduce load.

### Trace indexing
Indexing of traces are optimized by default. This reduces writes to Cassandra at the cost of memory
needed to cache state. This cache is tunable based on your typical trace duration and span count.

[Core annotations](https://github.com/openzipkin/zipkin/blob/master/zipkin/src/main/java/zipkin/Constants.java#L184), ex "sr", are not written to `annotations_index`, as they aren't intended for use in user queries. This significantly reduces
writes per trace.

### Over-fetching on Trace indexes
User-supplied query limits are over-fetched according to a configured index fetch multiplier in
attempts to mitigate redundant data returned from index queries.
