# Sharding

## About Sharding

Sharding is the technique of spreading your data across multiple database servers. It is difficult to add sharding to an existing application because it requires careful thought about the data model and data access.

### Scalability

For large data sets, sharding has the advantage of being more cost effective and more predictably scalable than a single massive server. It is hard to justify a massive database server purchase *today* to accommodate an unreliable growth forecast. Incrementally adding new database servers as demand grows is much a sounder financial approach. Virtualization and cloud technologies help alleviate this problem by making it easier to scale instances, but if you reach the limits of their instance scalability then you have the same problem.

### Disaster Recovery

Business continuity plans often specify a disaster recovery datacenter that can resume processing if the primary data center goes offline (usually due to a natural disaster like fire, earthquakes, flooding, etc.). Although this approach is common, it is usually plagued by two issues:

* The business must buy a complete data center that is nearly always idle
* Unless testing is unusually robust and frequent, there will always be doubt about whether the failover datacenter would be really able to assume a primary role

It would be immeasurably better to simply have both the primary *and* secondary datacenters actively processing transactions, each with enough reserve capacity to handle the load of the other in the event of failure. This negates both the waste of buying an idle datacenter and also any concerns about whether the failover site is truly ready to handle live transactions. In order for both datacenters to be simultaneously active, each one must “own” a segment of the data — which means data sharding.

### Global Availability

Your foreign customers will have a better, more responsive experience with your application if they access their data from a regionally nearby datacenter. Users accessing a single datacenter across the globe will experience noticeably slower connections. Using sharding with geo-replication can optimize regional access and still allow local queries across all the data.

Data privacy laws — particularly in China, Europe, and Russia — are also driving data storage to regional datacenters. A data sharding approach can be a useful way to consolidate the legally exportable subset of the data collected in these jurisdictions.

## Switching to Shards

If you are familiar with relational databases, you will discover that the database engine enforced some standard functionality that is no longer automatically available. For example, unique keys may not be unique across servers and foreign keys may refer to records that do not exist on that server. Thinking carefully through these issues will likely lead to successful workarounds.

ArgentSea offers essentially two services for managing sharded data:

* The [ShardSet](/api/ArgentSea.ShardSetsBase-2.ShardSet.html) unifies the many shard connections and directs queries to the correct shard and allows concurrent queries across all of them
* The [ShardKey](/api/ArgentSea.ShardKey-2.html) (and related [ShardChild](/api/ArgentSea.ShardChild-3.html)) are a “virtual compound key” that uniquely identifies a record using the shard Id and the record key.

ArgentSea’s querying architecture is designed to support concurrent queries across multiple shards. You can explore that further [here](../querying/querying.md).

Next: [The ShardKey and ShardChild](shardkey.md)
