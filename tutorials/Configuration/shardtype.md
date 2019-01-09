# The Shard Id Type

End each shard instance has a `ShardId` property, which identifies a specific subset (“shard”) of the data. ArgentSea uses a generic ShardId to allow you to define any data type you prefer.

This value is *critical* because it is not simply a key for a shard instance; the ShardId is generally used in combination with the record key to *uniquely identify* a record. ArgentSea identifies records in the shard set with a sort-of “virtual” compound key, consisting of both the shard identifier and the record key. Note that because records in a data shard may refer to foreign records in *other* shards, the “foreign shard” reference requires saving the shard identifier too. Consequently, the data type of the ShardId matters for all of the tables that hold the ShardId.

> [!WARNING]
> Once established, the ShardId *type* cannot be easily changed. The ShardId type is used in configuration, throughout your code, in the database, and across all shard sets. Make sure that you will not outgrow your ShardId type’s maximum value (nor unnecessarily require space that will never be used).
> If you are uncertain, consider using a `short` (Int16/SmallInt) data type for your ShardId.

> [!CAUTION]
> The shardId value is managed by the *client* when it calls a shard instance. If two clients are (mis-)configured differently (i.e. with different databases having the same shard Id) reading on one client and writing on the other client could result in data corruption. Always ensure that the shard Id configuration is consistent across all clients. It may be a good idea to include the shard Id with each query and validate it on the database server.

The JSON configuration ShardId type must correspond to whatever type you have defined for your application’s ShardId. If your application defines its ShardId as a string, then the JSON should be a string value (i.e. be in quotes); if a number, it should be numeric (i.e. a number, without quotes).

More details about the ShardId type is in the [Sharding](../sharding/sharding.md) section.

Next: [Resilience Strategies](resilience.md)
