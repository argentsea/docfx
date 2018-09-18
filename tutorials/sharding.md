# Sharding

Sharding is the technique of spreading your data across multiple database servers. For large data sets it has the advantage of being more cost effective than a single massive server. Also, it is more predictably scalable.

predictably scaleable

If you are familiar with relational databases, you will discover that the database engine enforced some standard functionality that is now not automatically available. For example, unique keys may not be unique across servers and foreign keys may refer to records that do not exist on that server. Thinking through these issues will likely lead to successful workarounds.

ArgentSea supports sharding with two objects: the `ShardSet` and the `ShardKey` (and related `ShardChild`). The ShardSet allows you to query on a particular shard (i.e. a specific connection) or across all shards in the set. The ShardKey (or ShardChild) identifies a unique record on a specific shard.

## ShardSets

## The ShardKey and ShardChild

### Data Origin

### MapShardKey attribute and MapShardChild attribute



Parameter copying
