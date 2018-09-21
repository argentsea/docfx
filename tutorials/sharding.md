# Sharding

## About Sharding

Sharding is the technique of spreading your data across multiple database servers. It is difficult to add sharding to an existing application because it requires careful thought about the data model and data access. However, the benefits of sharding are substantial, especially if you need to scale cost effectively.

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

* The [ShardSet](/api/ArgentSea.ShardDataStores-2.ShardDataSet.html) unifies the many shard connections and directs queries to the correct shard and allows concurrent queries across all of them
* The [ShardKey](/api/ArgentSea.ShardKey-2.html) (and related [ShardChild](/api/ArgentSea.ShardChild-3.html)) are a “virtual compound key” that uniquely identifies a record using the shard Id and the record key.

## The [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html)

All databases need a way to uniquely identify a record — a record key. With sharded data sets, a record key need to be unique across all the shards. Within a single database, uniqueness is easily managed; across a shard set, database engines can no longer enforce uniqueness for data they don’t know about. Additionally, on the client side, the query dispatcher needs to be able to use the record key in order to know to which shard connection to use.

There are two approaches to maintaining a unique key across multiple databases:

* __Use distinct identity ranges for each database in the shard set.__ The upside of this approach is that it is possible to combine data sets without conflicts; the downside is that configuration is complicated — on both the client and database servers — so mistakes are more likely, and some mistakes can be *very* hard to fix. The query dispatcher must know the various identity ranges hosted by each server in order to select the right connection.
* __Combine the *shard connection key* and the *record key* into a larger “compound key”.__ With this approach, finding the right shard connection is easy because the value is embedded in the compound record key. The database servers do not need to be configured with separate identity ranges, which in some case may allow smaller, more efficient key sizes (i.e. int vs bigint). Combining or splitting shards could be more complicated, however.

ArgentSea will work with either design. The [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) objects offer support for the second approach.

### Components

A [ShardKey](/api/ArgentSea.ShardKey-2.html) consists of three components: a *[DataOrgin](/api/ArgentSea.DataOrigin.html)*, a *ShardId*, and a *RecordId*. A [ShardChild](/api/ArgentSea.ShardChild-3.html) has the same values plus an additional *ChildId*.

#### The [DataOrgin](/api/ArgentSea.DataOrigin.html)

Both the [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) have a [DataOrigin](/api/ArgentSea.DataOrigin.html) value. The purpose of this value is to represent a data source. It is little more than a character value that you can choose to differentiate the data source.

For example, keys representing a *Customer* record might have a [DataOrgin](/api/ArgentSea.DataOrigin.html) of “c”, whereas keys representing a *Product* record might have a [DataOrgin](/api/ArgentSea.DataOrigin.html) of “p”. Because this simple tag identifies the data source, two different [ShardKeys](/api/ArgentSea.ShardKey-2.html) from the same shard and with the same record number will still not be equal because they represent different source data.

This capability is useful for helping prevent data from being accessed with the wrong type of key — like an inventory key inadvertently passed to fetch an account record. Also, this may be helpful for caching data, since you can use the same dictionary to cache objects of different types without key collision.

#### The ShardId

ArgentSea uses a generic type for the ShardId because the ideal data type will depend upon your requirements. Technically, the ShardId can be any of the types available to a RecordId (see below). Practically, however, it makes sense to avoid types without a corresponding SQL type and also avoid unnecessarily large data sizes. This leaves *byte*, *short*, *char* as the most storage-efficient choices; *int*, *string* are viable choices if your ShardId has other requirements — like needing to integrate with external systems.

If you really can’t decide and have no particular requirements, a simple starting place is to use *byte* if you have confidence that you will never need more than 256 shards in a [ShardSet](/api/ArgentSea.ShardDataStores-2.ShardDataSet.html), otherwise use *short*.

Because the ShardId value is used for the ShardSet configuration, for queries, and also for saving foreign shard references in your databases, this value cannot be easily changed once your project is established. The same ShardId type is used across all [ShardSets](/api/ArgentSea.ShardDataStores-2.ShardDataSet.html).

> [!NOTE]
> The database itself may not know what its own *ShardId* is. This sounds absurd until you realize that it is genuinely difficult to keep scores or hundreds of database schemas and procedures in sync while preserving a programmatic ShardId value. Your continuous delivery tooling will keep detecting any differences and trying to overwrite them. Fortunately, your connection *does* know this and can set the [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) values correctly.

#### The RecordId

Like the ShardId, the RecordId is also an generic type, which can be one of the following: 

| Possible Types |
| --- |
| *byte* |
| *char* |
| *DateTime*, |
| *DateTimeOffset* |
| *decimal* |
| *double*, |
| *float* |
| *Guid*, |
| *int* |
| *long* |
| *sbyte* |
| *short* |
| *string* |
| *TimeSpan* |
| *uint* |
| *ulong* |
| *ushort* |

If you have a data key that is not one of these types, the [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) objects will not know how to serialize the values. Unlike the ShardId, the data type of the RecordId can be diffeented for any table.

#### The ChildId

The [ShardChild](/api/ArgentSea.ShardChild-3.html) type gets its name from the parent-child relationship that is typical of a two-column compound key. The [ShardChild](/api/ArgentSea.ShardChild-3.html) includes the *RecordId* of the [ShardKey](/api/ArgentSea.ShardKey-2.html) along with a new generic *ChildId* value. A *ShardGrandChild* could also be created to support three-level compound record keys, but, so far, there hasn’t been demand for that.

The ChildId can be any of the types listed in the previous section and the data type can vary from table to table.

### Using The [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html)

Having a single object represent a compound record key adds only a little convenience. The real value comes from three capabilities: The shard Mapping attributes and the External key string.

#### ToString(), ToExternalString(), and FromExternalString()

Calling `ToString()` on a [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) returns a list of the constituent values.

The `ToExternalKey()` function serializes the [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) values into a URL-safe string. This string also has a small amount of tampering protection.

As you would expect, the `FromExternalString()` function reverses the operation, returning a ShardKey or ShardChild instance from a valid string.

The *External String* value can be used with REST endpoints and the like to specify a sharded record using a single argument. 

#### The [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) and [MapShardChild](/api/ArgentSea.MapShardChildAttribute.html) Attributes

The [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) and [MapShardChild](/api/ArgentSea.MapShardChildAttribute.html) attributes map the shard information, record key, and (as appropriate) the child record value to a new [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) instance respectively.

The simplest implementation is to simply add the [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) or [MapShardChild](/api/ArgentSea.MapShardChildAttribute.html) attribute and the type-appropriate *MapTo* attribute(s).

## [SQL Server](#tab/tabid-sql)

````C#
[MapShardKey('c', "@CustomerId")]
[MapToSqlInt("@CustomerId")]
public ShardKey<byte, int> CustomerKey { get; set; }

````

## [PostgreSQL](#tab/tabid-pg)

````C#
[MapShardKey('c', "CustomerId")]
[MapToPgInteger("CustomerId")]
public ShardKey<byte, int> CustomerKey { get; set; }
````

***

This example sets the property to a [ShardKey](/api/ArgentSea.ShardKey-2.html) instance with a *[DataOrgin](/api/ArgentSea.DataOrigin.html)* of “c”, the *ShardId* to the value of the data connection, and the *RecordId* the “CustomerId” column or parameter value.

The [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) attribute’s first argument can be either a [DataOrgin](/api/ArgentSea.DataOrigin.html) instance or a char from which a [DataOrgin](/api/ArgentSea.DataOrigin.html) will be created. 

The second argument is the name of the data parameter or column. This name must *exactly* match the name in the data *MapTo* attribute.

The [MapShardChild](/api/ArgentSea.MapShardChildAttribute.html) attribute is nearly identical, except for the additional *ChildId* parameter:

## [SQL Server](#tab/tabid-sql)

````C#
[MapShardChild('O', "@OrderId", "@OrderItemId")]
[MapToSqlBigInt("@OrderId")]
[MapToSqlSmallInt("@OrderItemId")]
public ShardChild<byte, long, short> OrderItemKey { get; set; }

````

## [PostgreSQL](#tab/tabid-pg)

````C#
[MapShardChild('O', "OrderId", "OrderItemId")]
[MapToPgBigint("OrderId")]
[MapToPgSmallint("OrderItemId")]
public ShardChild<byte, long, short> OrderItemKey { get; set; }
````

***

In both previous examples, the ShardId will be implicitly obtained from the connection’s ShardId. Occasionally, however, particularly when a data record refers to a record in a foreign shard, the *ShardId* must come from the database record. To do this, just add a *ShardID* parameter and *MapTo* attribute:

## [SQL Server](#tab/tabid-sql)

````C#
[MapShardChild('O', "@OrderShardId", "@OrderId", "@OrderItemId")]
[MapToSqlTinyInt("@OrderShardId")]
[MapToSqlBigInt("@OrderId")]
[MapToSqlSmallInt("@OrderItemId")]
public ShardChild<byte, long, short> OrderItemKey { get; set; }

````

## [PostgreSQL](#tab/tabid-pg)

````C#
[MapShardChild('O', "OrderShardId", "OrderId", "OrderItemId")]
[MapToPgTinyint("OrderShardId")]
[MapToPgBigint("OrderId")]
[MapToPgSmallint("OrderItemId")]
public ShardChild<byte, long, short> OrderItemKey { get; set; }
````

***

#### Equals

#### Empty

## ShardSets

The root injectable service is a `ShardSets` object, which is simply a collection of ShardSets. You can reference any ShardSet by name. Many (probably most) applications will have only one ShardSet, but this supports circumstances where different data is sharded differently. Your user information, for example, might be distributed across global datacenters, while product availability information might be sharded for business process reasons.

You can access a ShardSet by name:

```C#
    public class SubscriberStore
    {
        private readonly SqlDatabases _dbs;
        private readonly SqlShardSets<byte>.ShardDataSet _shardSet;

        private readonly ILogger<SubscriberStore> _logger;
        public SubscriberStore(SqlShardSets<byte> shardSets, ILogger<SubscriberStore> logger)
        {
            _shardSet = shardSets.ShardSets["Subscribers"];
            _logger = logger;
        }

```


 might be For example, you could have a 

A ShardSet is a collection

## Virtual Compound Keys


### Data Origin

### MapShardKey attribute and MapShardChild attribute



Parameter copying
