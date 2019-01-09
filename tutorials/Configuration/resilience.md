# Resilience Strategies

Unexpected failures happen, and business-critical applications must be able to recover gracefully. ArgentSea uses [Polly](http://www.thepollyproject.org/) to offer a combination of retry logic and circuit breaking.

The properties specific to a resilience strategy are:

````json
{
  "CircuitBreakerFailureCount": 20,
  "CircuitBreakerTestInterval": 5000,
  "ConnectTimeout": 5,
  "RetryCount": 6,
  "RetryInterval": 256,
  "RetryLengthening": "Fibonacci",
}
````

If no retry or circuit breaking properties are configured, ArgentSea uses a default resilience strategy using automatic retries and circuit breaking. Like everything else, these values participate in the ArgentSea Hereditary Configuration Hierarchy — you can set these values globally (most likely) or at any level down to the individual Read or Write connection.

## Retries

Retries occur on errors that are defined as *transient*. A network interruption may quickly resolve itself, so it makes sense to retry after a short interval; it doesn’t make sense to retry after a permission exception. (The list of errors defined as *transient* is in the provider-specific implementation of IDataProviderServiceFactory. You can view this in the source code).

The properties that define the retry behavior are:

* The `RetryCount` setting determines how many times the connection retries before aborting and raising an error back to the caller.
* The `RetryInterval` determines the length of time (in milliseconds) between retries.
* The `RetryLengthening` value can add an additional pause between subsequent retries.

Presumably, if the system encounters a transient error, it should retry quickly, then, if the retry is not successful, it should wait a bit longer for the error to clear before retrying again. The `RetryLengthening` value is what determines how much longer it will pause on subsequent retries before giving up.

The [Retry Sequence Lengthening](/api/ArgentSea.DataResilienceConfiguration.SequenceLengthening.html) values are:

* __Linear__ - each retry is the same duration as specified in `RetryInterval`
* __Fibonacci__ - The first retry is at `RetryInterval`, each subsequent retry interval pauses for the duration of the previous two combined.
* __HalfSquare__ - the retry count number is squared, then divided by two, then multiplied by `RetryInterval`
* __Squaring__ - each retry attempt doubles the duration of the previous one.

You can visualize the impact of `RetryLengthening` with these charts:

![Linear](../../images/retrygraphs/linear.jpg)

![HalfSquare](../../images/retrygraphs/halfsquare.jpg)

![Fibonacci](../../images/retrygraphs/fibonacci.jpg)

![Doubling](../../images/retrygraphs/doubling.jpg)

If a Resilience Strategy is not defined, ArgentSea will use a default strategy. Currently, this is:

| Setting | Default Value |
| --- | --- |
| RetryCount | 6 tries |
|RetryInterval | 256 milliseconds |
| Lengthening | Fibonacci |
| (Connect) Timeout | 5 |

> [!NOTE]
> SQL Server’s ADO.NET provider also offers automatic retries. ArgentSea disables this in lieu of its own functionality, which logs these automatic retries. This valuable environment diagnostic information should not be invisible.

### Connection Timeout

The connection timeout value is critical to determining the duration of connection attempts before failure. The ADO.NET default of 15 seconds is far too long, so the ArgentSea default is 2 seconds. Datacenter connections are generally resolved in that time unless something is wrong. If you have a WAN or high-latency connection, you should consider increasing this value.

It is not guaranteed that any defaults will remain unchanged in future versions.

Note that a high `RetryCount` and/or connection timeout could create a very long delay before a connection is allowed to ultimately fail.

## Circuit Breaking

When a database connection is unavailable, this can cause serious downstream problems. Processes may pile-on further requests even while earlier requests are simply waiting
to time out. As this continues, the queue of backlogged requests becomes so large that the caller itself can manage no more. The bottleneck will then start blocking other systems too. What started as a broken connection to a single database eventually becomes fatal to the entire system!

This is the reason to add a “circuit breaker” — a fail-fast mechanism to ensure that callers do not wait needlessly for queued connections that are unlikely to succeed, and which are blocking other processes too.

Once the circuit breaker is tripped, subsequent connections will fail *immediately*. This prevents queuing, bottleneck blocking, and downstream failures. While tripped, the circuit breaker will periodically allow a single transaction to proceed; if it successful the circuit breaker is reopened. In this way, a system restoration will automatically close the circuit breaker too so that connections can resume.

The `CircuitBreakerFailureCount` value determines how many sequential failures will trigger the circuit breaker. The `CircuitBreakerTestInterval` value determines how often (in milliseconds) the circuit breaker will allow a single transaction through.

## Data Failover for High-Availability and Disaster Recovery

ArgentSea itself does not currently include specific functionality that enables an automatic failover to a standby database server. This is not to say that high-availability or disaster-recovery solutions cannot be used, only that ArgentSea is not opinionated about whatever approach you choose.

Often, failover is managed through DNS changes or via configuration of the .NET data provider (such as the *FailoverPartner* property), which may not even require client connection string changes. It would be difficult to build a robust strategy that predictably worked well with the variations of approaches possible. Furthermore, business continuity plans typically do not expect disaster recovery plans to be fully “automatic” at the individual client level; a lot of infrastructure must be coordinated in a robust failover, and a rogue client should not failover without a coordinated signal.

Consequently, ArgentSea makes it *possible* to easily build failover capability, but does not natively offer this. All of the connection properties in the configuration hierarchy are updatable. This allows you to build failover logic that updates the connection information — server endpoints, database names, etc. — given whatever trigger you prefer.

> [!NOTE]
> Once created, both the [ShardSets](https://docs.argentsea.com/api/ArgentSea.ShardSetsBase-2.DataConnection.html) and [Databases](https://docs.argentsea.com/api/ArgentSea.DatabasesBase-1.html) singleton collections themselves are immutable, although any connection property in the configuration hierarchy can still be updated. In other words, after the Hereditary Configuration Hierarchy is created, you cannot change the layout of the hierarchy, but the connection properties of the members can still be changed and will update the client connections. Child objects will also continue to inherit any updates from their parents. Of course, updating connection properties is not fully thread-safe, but it will not impact queries that have already started.

The configuration objects are [.NET Options](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options?view=aspnetcore-2.1) classes, which are obtained through an `IOptions<>` injection. One possible point of confusion is that the you reference objects in the [ShardSets](https://docs.argentsea.com/api/ArgentSea.ShardSetsBase-2.DataConnection.html) and [Databases](https://docs.argentsea.com/api/ArgentSea.DatabasesBase-1.html) *collections* by their (string) key; however, the Options objects themselves originate as *arrays* that reflect your configuration layout. You must use an integer index to reference a shard set or connection. The value of this index depends on the order in your configuration files and the order in which they are loaded. In other words, you can verify the configuration options object by checking the key property, but you cannot use the key as an indexer.

Next: [Loading the Configuration](loading.md)
