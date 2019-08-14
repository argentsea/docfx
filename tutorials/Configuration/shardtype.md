# The Shard Id Type

End each shard instance has a `ShardId` property, which identifies a specific subset (“shard”) of the data. This value is not simply a key for a shard instance; the ShardId is generally used in combination with the record key to *uniquely identify* a record. ArgentSea identifies records in the shard set with a sort-of “virtual” compound key, consisting of both the numeric shard identifier and the record key. Note that because records in a data shard may refer to foreign records in *other* shards, the “foreign shard” reference requires saving the shard identifier too.


> [!CAUTION]
> The shardId value is managed by the *client* when it calls a shard instance. If two clients are (mis-)configured differently (i.e. with *different* databases having the *same* shard Id) reading on one client and writing on the other client could result in data corruption. Always ensure that the shard Id configuration is consistent across all clients! You might consider including the shard Id with each query and validating it on the database server.

More details about the ShardId type is in the [Sharding](../sharding/sharding.md) section.

> [!NOTE]
> The initial version of ArgentSea used a generic type as the ShardId — allowing the application to use an integer, string, short, etc. to identify each shard. This flexibility created an unnecessarily complex object model and redundant and verbose client code, particularly since the generic type could not change within the application.
> Even then, he best choice for the ShardId type was clearly a `short` (Int32/SmallInt): it is a small enough for efficient storage, yet large enough to theoretically support 65,535 shards.
> The current version of ArgentSea assumes this type and is much easier to use as a result.

Next: [Resilience Strategies](resilience.md)
