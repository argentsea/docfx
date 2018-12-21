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

ArgentSea’s querying architecture is designed to support concurrent queries across multiple shards. You can explore that further [here](querying.md).

## The [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html)

All databases need a way to uniquely identify a record — a record key. With sharded data sets, a record key need to be unique across all the shards. Within a single database, uniqueness is easily managed; across a shard set, database engines can no longer enforce uniqueness for data they don’t know about. Additionally, on the client side, the query dispatcher needs to be able to use the record key in order to know to which shard connection to use.

There are two approaches to maintaining a unique key across multiple databases:

* __Use distinct identity ranges for each database in the shard set.__ The upside of this approach is that it is possible to combine data sets without conflicts; the downside is that configuration is complicated — on both the client and database servers — so mistakes are more likely, and some mistakes can be *very* hard to fix. The query dispatcher must know the various identity ranges hosted by each server in order to select the right connection.
* __Combine the *shard connection key* and the *record key* into a larger “compound key”.__ With this approach, finding the right shard connection is easy because the value is embedded in the compound record key. The database servers do not need to be configured with separate identity ranges, which in some case may allow smaller, more efficient key sizes (i.e. int vs bigint). Combining or splitting shards could be more complicated, however.

ArgentSea will work with either design. The [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) objects offer support for the second approach.

### Components

A [ShardKey](/api/ArgentSea.ShardKey-2.html) consists of three components: a *[DataOrigin](/api/ArgentSea.DataOrigin.html)*, a *ShardId*, and a *RecordId*. A [ShardChild](/api/ArgentSea.ShardChild-3.html) has the same values plus an additional *ChildId*.

<table border="0" margin="0" padding="0"><tr><td width="43%"><img src="/images/shardkey.svg"></td><td width="57%"><img src="/images/shardchild.svg"></td></tr></table>

#### The [DataOrigin](/api/ArgentSea.DataOrigin.html)

Both the [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) have a [DataOrigin](/api/ArgentSea.DataOrigin.html) value. The purpose of this value is to represent a data source. It is little more than a character value that you can choose to differentiate the data source.

For example, keys representing a *Customer* record might have a [DataOrigin](/api/ArgentSea.DataOrigin.html) of “c”, whereas keys representing a *Product* record might have a [DataOrigin](/api/ArgentSea.DataOrigin.html) of “p”. Because this simple tag identifies the data source, two different [ShardKeys](/api/ArgentSea.ShardKey-2.html) from the same shard and with the same record number will still not be equal because they represent different source data.

> [!IMPORTANT]
> One [DataOrigin](/api/ArgentSea.DataOrigin.html) character value is reserved: “0” (Unicode character Zero, Unicode numeric value 30).
>
> This is used for the [DataOrigin](/api/ArgentSea.DataOrigin.html) of `ShardKey.Empty` and `ShardChild.Empty`. Creating a [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) with a “zero” [DataOrigin](/api/ArgentSea.DataOrigin.html) character but non-default (i.e. not zero or not null) *ShardId* or *RecordId* values will throw an [InvalidShardArgumentsException](/api/ArgentSea.InvalidShardArgumentsException.html) error.

This capability is useful for helping prevent data from being accessed with the wrong type of key — like an inventory key inadvertently passed to fetch an account record. Also, this may be helpful for caching data, since you can use the same dictionary to cache objects of different types without key collision.

#### The ShardId

The ShardId is used to identify a particular shard in the ShardSet. The core ArgentSea framework uses a generic type for the ShardId because the ideal data type will depend upon your requirements. Technically, the ShardId can be any of the types available to a RecordId (see below). Practically, however, it makes sense to avoid types without a corresponding SQL type and also avoid unnecessarily large data sizes. This leaves *byte* (SQL Server only), *short*, *char* as the most storage-efficient choices; *int*, *string* are viable choices if your ShardId has other requirements — like needing to integrate with external systems.

In essence, the most efficient ShardId type for SQL Server is byte/Tinyint, and for PostgreSQL is Int16(short)/Smalllint.

If you really can’t decide and have no particular requirements, a simple starting place is to use *byte* if are using SQL Server and you have confidence that you will never need more than 256 shards in a [ShardSet](/api/ArgentSea.ShardSetsBase-2.ShardSet.html), otherwise start with *short*.

Because the ShardId value is used in configuration, queries, and also for saving foreign shard references in your databases, once your project is established this value cannot be easily changed. The same ShardId type is used across all [ShardSets](/api/ArgentSea.ShardSetsBase-2.html).

> [!NOTE]
> The database itself may not know what its own *ShardId* is. This sounds absurd until you realize that it is genuinely difficult to keep scores or even hundreds of database schemas and procedures in sync while preserving a programmatic ShardId value. Your continuous delivery tooling will keep detecting any differences and trying to overwrite them. Fortunately, your connection *does* know this and can set the [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) values correctly.

#### The RecordId

Like the ShardId, the RecordId is also an generic type, which can be one of the following:

| RecordId (and ChildId) Possible Data Types |
|--- |
| *byte*, *char*, *DateTime*, *DateTimeOffset*, *decimal*, *double*, *float*, *Guid*, *int*, *long*, *sbyte*, *short*, *string*, *TimeSpan*, *uint*, *ulong*, *ushort* |

If you have a data key that is not one of these types, the [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) objects will not know how to serialize the values.

Unlike the ShardId, the data type of the RecordId (and/or ChildId) need not be universal; it *can* be different for each table.

#### The ChildId

The [ShardChild](/api/ArgentSea.ShardChild-3.html) type gets its name from the parent-child relationship that is typical of a two-column compound key. The [ShardChild](/api/ArgentSea.ShardChild-3.html) includes the *RecordId* of the [ShardKey](/api/ArgentSea.ShardKey-2.html) along with a new generic *ChildId* value. A *ShardGrandChild* could also be created to support three-level compound record keys, but, so far, there hasn’t been demand for that.

The ChildId can be any of the types listed in the previous section and the data type can also vary from table to table.

### Using The [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html)

Having a single object represent a compound record key adds only a little convenience. The real value comes from three capabilities: The shard Mapping attributes and the External key string.

#### ToString(), ToExternalString(), and FromExternalString()

Calling `ToString()` on a [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) returns a list of the constituent values.

The `ToExternalKey()` function serializes the [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) values into a URL-safe string. This string also has a small amount of tampering protection.

As you would expect, the `FromExternalString()` function reverses the operation, returning a ShardKey or ShardChild instance from a valid string.

The *External String* value can be used with, say, REST endpoints to specify a sharded record using a single argument.

#### The [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) and [MapShardChild](/api/ArgentSea.MapShardChildAttribute.html) Attributes

The [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) and [MapShardChild](/api/ArgentSea.MapShardChildAttribute.html) attributes map the shard information, record key, and (as appropriate) the child record value to a new [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) instance respectively.

The simplest implementation is to simply add the [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) or [MapShardChild](/api/ArgentSea.MapShardChildAttribute.html) attribute and the type-appropriate *MapTo* attribute(s).

## [SQL Server](#tab/tabid-sql)

````csharp
[MapShardKey('c', "@CustomerId")]
[MapToSqlInt("@CustomerId")]
public ShardKey<byte, int> CustomerKey { get; set; }

````

## [PostgreSQL](#tab/tabid-pg)

````csharp
[MapShardKey('c', "CustomerId")]
[MapToPgInteger("CustomerId")]
public ShardKey<short, int> CustomerKey { get; set; }
````

***

This example sets the property to a [ShardKey](/api/ArgentSea.ShardKey-2.html) instance with a *[DataOrigin](/api/ArgentSea.DataOrigin.html)* of “c”, the *ShardId* to the value of the data connection, and the *RecordId* the “CustomerId” column or parameter value.

The [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) attribute’s first argument can be either a [DataOrigin](/api/ArgentSea.DataOrigin.html) instance or a char from which a [DataOrigin](/api/ArgentSea.DataOrigin.html) will be created.

The second argument is the name of the data parameter or column. This name must *exactly* match the name in the data *MapTo* attribute.

The [MapShardChild](/api/ArgentSea.MapShardChildAttribute.html) attribute is nearly identical, except for the additional *ChildId* parameter:

## [SQL Server](#tab/tabid-sql)

````csharp
[MapShardChild('O', "@OrderId", "@OrderItemId")]
[MapToSqlBigInt("@OrderId")]
[MapToSqlSmallInt("@OrderItemId")]
public ShardChild<byte, long, short> OrderItemKey { get; set; }

````

## [PostgreSQL](#tab/tabid-pg)

````csharp
[MapShardChild('O', "OrderId", "OrderItemId")]
[MapToPgBigint("OrderId")]
[MapToPgInteger("OrderItemId")]
public ShardChild<short, long, int> OrderItemKey { get; set; }
````

***

In both previous examples, the ShardId will be *implicitly* obtained from the connection’s ShardId. In the case of results that include then primary key column, this works well. However, when a data record references the primary key of a sharded table, the *ShardId* of the ShardKey or ShardChild must *explicitly* come from the database record. To do this, just add a *ShardID* parameter to the *MapShard* attribute and the additional *MapTo* data attribute:

## [SQL Server](#tab/tabid-sql)

````csharp
[MapShardKey('c', "@CustomerShardId", "@CustomerId")]
[MapToSqlTinyInt("@CustomerShardId")]
[MapToSqlInt("@CustomerId")]
public ShardKey<byte, int> CustomerKey { get; set; }

[MapShardChild('O', "@OrderShardId", "@OrderId", "@OrderItemId")]
[MapToSqlTinyInt("@OrderShardId")]
[MapToSqlBigInt("@OrderId")]
[MapToSqlSmallInt("@OrderItemId")]
public ShardChild<byte, long, short> OrderItemKey { get; set; }

````

## [PostgreSQL](#tab/tabid-pg)

````csharp
[MapShardKey('c', "CustomerId")]
[MapToSqlTinyint("CustomerShardId")]
[MapToPgInteger("CustomerId")]
public ShardKey<short, int> CustomerKey { get; set; }

[MapShardChild('O', "OrderShardId", "OrderId", "OrderItemId")]
[MapToPgSmallint("OrderShardId")]
[MapToPgBigint("OrderId")]
[MapToPgSmallint("OrderItemId")]
public ShardChild<short, long, short> OrderItemKey { get; set; }
````

***

### Null Values

Because both [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) are structs, a variable or property of this type *cannot* be null. [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) objects are initialized to ShardKey.Empty or ShardChild.Empty respectively.

If a [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) represents a database field that might be Null, the [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) property or variable should be wrapped in the `Nullable<>` type. The *MapTo* attribute will set the `Nullable<ShardKey<>>` or `Nullable<ShardChild<>>` property to null if any of the constituent database column values are Null. If the underlying type is not `Nullable<>` and the database value is Null, the Mapper with throw an error (except as described in the next paragraph).

In most cases, a [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) represents a primary key, so a database Null value really represents a non-existent record. In this case, the desired behavior is probably to return the entire parent object as null. Marking the *MapTo* attribute(s) as *required* implements this behavior. When the *required* parameter is set, the [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) property does not need to be `Nullable<>` since a Null database value will return a null result object.

# ShardSets

A “shard set” is a collection of databases with essentially identical schemas, each of which contain a segment of the data. Many — probably most — sharded applications will have only one ShardSet, but this supports contexts where multiple sharding plans exist. For example, User information might be sharded globally by datacenter location, while product availability information might be sharded by subsidiary (ok, this specious example might be better served via microservices; the point is that the framework does not preclude multiple ShardSets if you need them).

The root injectable service is a [ShardSets](/api/ArgentSea.ShardSetsBase-2.html) object, which is merely a collection of `ShardSet` instances.

## The ShardSets Class Hierarchy

The [ShardSets](/api/ArgentSea.ShardSets-2.html) collection is the root of an object hierarchy. The child objects in the hierarchy are implemented as nested classes. This simplifies the implementation, but can also make declarations somewhat verbose.

![ShardSetHierarchy](../images/shardsets.svg)

### Nested classes

* `ShardSets` - the root collection, which provides access to any of the various sharding schemas.
* `ShardSets.ShardSet` - a collection of servers which have the same schema and different segments of data.
* `ShardSets.ShardInstance` - a shard (single data store) with one segment of data. Includes (optionally) separate read and write connections.
* `ShardSets.DataConnection` - A database connection to a shard.

## Accessing the ShardSets

In .NET Core, the ShardSets collection is an injectable service. The instructions in the [Configuration](configuration.md) section can help you with setup. You can reference any ShardSet by name (i.e. a string key), which is also defined during configuration. Note that the key name is case/accent/kana sensitive; it must exactly match the value used in your configuration.

Because it is unlikely that you would need to access more than one ShardSet in the same data access class, your class-level variable should capture only the relevant ShardSet. You can access a ShardSet by name (i.e. a string key value):

## [SQL Server](#tab/tabid-sql)

```csharp
    public class SubscriberStore
    {
        private readonly SqlShardSets<string>.ShardSet _shardSet;
        private readonly ILogger<SubscriberStore> _logger;

        public SubscriberStore(SqlShardSets<string> shardSets, ILogger<SubscriberStore> logger)
        {
            _shardSet = shardSets["Subscribers"];
            _logger = logger;
        }

```

## [PostgreSQL](#tab/tabid-pg)

```csharp
public class SubscriberStore
{
    private readonly PgShardSets<string>.ShardSet _shardSet;
    private readonly ILogger<SubscriberStore> _logger;

    public SubscriberStore(PgShardSets<string> shardSets, ILogger<SubscriberStore> logger)
    {
        _shardSet = shardSets["Subscribers"];
        _logger = logger;
    }
}
```

***

## Querying a ShardSet

There are two types of ShardSet queries:

* *Queries on a particular shard* - usually to obtain a specific record, like when you have a ShardKey.
* *Queries across all shards* - when you need a combined list or when don’t know the specific shard(s) to search.

### Accessing a Shard

Access any shard in the ShardSet collection using a shardId key value, just like you would with any other collection. The ShardId value often comes from the ShardId property of a `ShardKey` or `ShardChild`; for convenience, you can simply provide the `ShardKey` or `ShardChild` object instead.

```csharp
/// all of these are equally valid:
var shard = myShardSet[myShardId];
var shard = myShardSet[myShardKey.ShardId];
var shard = myShardSet[myShardKey];
var shard = myShardSet[myShardChild];
```

If you have implemented a solution using identity ranges, just call your custom resolver to get the shard index.

### The Default Shard

When your data clients need to insert a new record, they need to know which shard within the ShardSet to put it in. If, for example, your shards are segmented by region, your regional clients should “default” to the appropriate shard when creating new records. This is configured by the `DefaultShardId` property in your ShardSet configuration. 

The default shard works exactly like any other shard, except that you do not need to specify a collection key; instead you can get it from the `DefaultShard` property.

```csharp
var shard = myShardSet.DefaultShard;
```

### Shard Connections

Each [shard](/api/ArgentSea.ShardSetsBase-2.ShardInstance.html) has two data connections, exposed as `Read` property and a `Write` property. The `Read` and `Write` connection properties correspond to the read and write connections defined in your connection [configuration](configuration.md). If you have both connections defined in your configuration, then the query will execute on the corresponding read or write connection; if only Read or Write is configured, it doesn’t matter which you use since they will both have the same connection.

## [SQL Server](#tab/tabid-sql)

```csharp
public async Task<Subscriber> GetSubscriber(ShardKey<byte, int> subscriberKey, CancellationToken cancellation)
{
    var prms = new QueryParameterCollection()
        .AddSqlIntInputParameter("@SubId", subscriberKey.RecordId);
    return await _shardSet[subscriberKey].Read.MapOutputAsync<Subscriber>("ws.GetSubscriber", prms, cancellation);
}
```

## [PostgreSQL](#tab/tabid-pg)

```csharp
public async Task<Subscriber> GetSubscriber(ShardKey<short, int> subscriberKey, CancellationToken cancellation)
{
    var prms = new QueryParameterCollection()
        .AddPgIntegerInputParameter("SubId", subscriberKey.RecordId);
    return await _shardSet[subscriberKey].Read.MapOutputAsync<Subscriber>("ws.GetSubscriber", prms, cancellation);
}
```

***

Several database implementations — such as *SQL Server Availability Groups* or *AWS Aurora PostgreSQL* to name a couple of examples — enable a master server to handle both reads and writes and separate clone instances that can handle read-only traffic. Most applications have a greater percentage of reads than writes, so this is a great way to scale-out database access. However, there are two issues of concern:

* ArgentSea has *no idea* which stored procedures or functions update data and which are read-only, so it is left to the application developer to designate this by selecting  the appropriate connection property (Read or Write).
* There is often some latency between the time that data is saved and when it is available from the read instance. This temporary data inconsistency can cause problems or confusion due to missing data.

There are several architectural solutions to the latency-driven data inconsistency problem, such as intelligent caching, client observable collections, delayed retries, and retries on the Write connection. Due to the variations in environments, optimal solutions, and the challenge of simple determining when a missing record is really expected, ArgentSea does not attempt an automatic retry on the Write connection.

To implement your own latency handling, you can easily implement an automatic retry using the Write connection after an unexpectedly missing record on the Read connection. In this example method we retrieve data by key value, so a missing record is unexpected and might be due to replication latency. The code assumes that the subscriber key has the “required” attribute set so that the Mapper returns a null object if the key is null. The resolution is to simply retry on the Write connection.

```csharp
    var sub = await _shardSet[subscriberKey].Read.MapReaderAsync<Subscriber>("ws.GetSubscriber", prms, cancellation);
    // add automatic retry on write connection if subscriber is not found.
    if (sub is null)
    {
        // consider logging the retry on the write connection
        var sub = await _shardSet[subscriberKey].Write.MapReaderAsync<Subscriber>("ws.GetSubscriber", prms, cancellation);
    }
    return sub;
}

```

> [!TIP]
> Even if you are not using a scale-out strategy today, it would be a good idea to use the `Read` and `Write` properties as if you were. This would make a future migration to separate read and write instances a little easier.  
> You might also consider using different database schemas for read-only and write-capable procedures or functions. This helps underline the importance of separating read-only activity to your data developers. And testing may be easier if each connection’s permissions is limited to the appropriate schema.


### Shard Query Methods

There are several query methods, described briefly below and in more detail in the [querying](querying.md) tutorial. The arguments for these query methods are described in the next section.

#### *RunAsync*

Executes a stored procedure without returning a result — other than an Exception if it is not successful. Presumably, this method would only be called on the Write connection but nothing prevents running a procedure on the Read connection. This method is available on individual shards, but not on the entire ShardSet.

#### *LookupAsync*

Executes a stored procedure and returns the value (string, number, etc.) of either the return result or (single) output parameter. This method useful to fetch a single value from the shard rather than an entire record. This method is also available on individual shards, but not on the entire ShardSet.

#### *ListAsync*

Executes a stored procedure and returns a list containing a Model object, one entry for each record in the result set.

The objects are created using Mapping attributes. If the Model object does not have attributes, you can create a List using QueryAsync with a custom handler.

This method is available on both individual shards and the entire ShardSet. Results across ShardSets are combined into a single list.

#### *QueryAsync*, *QueryFirstAsync*, and *QueryAllAsync*

Executes a stored procedure and returns a (potentially complex) result object from output parameters and/or result sets. The method can create an arbitrary result (List, Dictionary, Model, etc.) via a custom delegate that constructs the response.

#### *MapOutputAsync*, *MapOutputFirstAsync*, and*MapOutputAllAsync*

Uses the Mapper to build a result using output parameters. The Mapper can use DataReader results to build list properties. MapOutputAsync is found on individual shards; MapOutputFirstAsync and MapOutputAllAsync are on ShardSets and return the first non-null result, or a list of all non-null results, respectively.

#### *MapReaderAsync*, *MapReaderFirstAsync*, and *MapReaderAllAsync*

Also uses the Mapper to build a results, but does so through a single-row DataReader result, rather than output parameters. List properties of the object result can also be populated through additional result sets.

> [!NOTE]
> Parallelized queries across a ShardSet use the Read connection. Writes should be managed on individual shards.

### Arguments

#### Procedure (required)

The name of the stored procedure, including the schema name.

#### Parameters (required)

In most cases this should be a [QueryParametersCollection](/api/ArgentSea.QueryParameterCollection.html) object. Technically, this argument can be any parameter collection, but the collections provided by ADO.NET are problematic: the DbParameterCollection is abstract, while the provider implementations (SqlParameterCollection and NpgsqlParameterCollection) can only be created by existing command objects.

#### shardParameterOrdinal (optional)

This parameter allows you to set a parameter to current ShardId value.

For example, you might want to return a list of related records that do not exist on the current shard, but the database itself does not know its own shard number. Or perhaps the database *does* know its ShardId and, because mixing up ShardIds in your configuration would be catastrophic, you want to validate that the expected ShardId on the connection corresponds to the ShardId of the database (a practice that I follow).

If set the argument to a value of zero or higher, ArgentSea will assign parameter at that (zero-based) index the value of the connection’s ShardId. If set to -1, no parameter will be assign a ShardId value.

#### cancellationToken

The cancellation token is used to cancel the query on all threads. Typically, you would pass the cancellation token from your MVC web method.

#### resultHandler (optional)

The QueryAsync method requires a method that knows how to convert the data results (output parameters and/or DataReader results) into an object instance. The result could be a Model, List, Dictionary, etc.

The handler must have a method signature corresponding to the [QueryResultModelHandler](/api/ArgentSea.QueryResultModelHandler-3.html) delegate.

Because the ArgentSea Mapper includes method signature that can act as a [QueryResultModelHandler](/api/ArgentSea.QueryResultModelHandler-3.html) delegate. The query methods that do not require this parameter assume the Mapper is being used. The generic result type must implement *MapTo* property attributes for the Mapper to function.

#### TopOne

Set this argument `True` if only one result is expected. For example, suppose you are searching a ShardSet for a User account matching a login. There should only be one match, so as soon as the first match is obtained you want to return the result object and abandon any remaining queries.

Technically, when this argument is True, ArgentSea checks each shard query to see if it has a non-null Model result. If it finds one, it fires the cancellation token for any shard connection that has not yet completed, and returns the result.

Of course, if the search conditions are not unique (which is difficult to enforce with sharded data), any duplicate result(s) will be lost.
