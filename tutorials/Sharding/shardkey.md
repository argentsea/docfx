# The [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html)

All databases need a way to uniquely identify a record — a record key. With sharded data sets, a record key need to be unique across all the shards. Within a single database, uniqueness is easily managed; across a shard set, database engines can no longer enforce uniqueness for data they don’t know about. Additionally, on the client side, the query dispatcher needs to be able to use the record key in order to know to which shard connection to use.

There are two approaches to maintaining a unique key across multiple databases:

* __Use distinct identity ranges for each database in the shard set.__ The upside of this approach is that it is possible to combine data sets without conflicts; the downside is that configuration is complicated — on both the client and database servers — so mistakes are more likely, and some mistakes can be *very* hard to fix. The query dispatcher must know the various identity ranges hosted by each server in order to select the right connection.
* __Combine the *shard connection key* and the *record key* into a larger “compound key”.__ With this approach, finding the right shard connection is easy because the value is embedded in the compound record key. The database servers do not need to be configured with separate identity ranges, which in some case may allow smaller, more efficient key sizes (i.e. int vs bigint). Combining or splitting shards could be more complicated, however.

ArgentSea will work with either design. The [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) objects offer support for the second approach.

## Components

A [ShardKey](/api/ArgentSea.ShardKey-2.html) consists of three components: a *DataOrigin* char value, a *ShardId*, and a *RecordId*. A [ShardChild](/api/ArgentSea.ShardChild-3.html) has the same values plus an additional *ChildId*.

<table border="0" margin="0" padding="0"><tr><td width="43%"><img src="/images/shardkey.svg"></td><td width="57%"><img src="/images/shardchild.svg"></td></tr></table>

### The DataOrigin

Both the [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) have a *DataOrigin* value. The purpose of this value is to represent a data source. It is simply a character value that you can choose to differentiate the data source.

For example, keys representing a *Customer* record might have a *DataOrigin* of “c”, whereas keys representing a *Product* record might have a *DataOrigin* of “p”. Because this simple tag identifies the data source, two different [ShardKeys](/api/ArgentSea.ShardKey-2.html) from the same shard and with the same record number will still not be equal because they represent different source data.

> [!IMPORTANT]
> One *DataOrigin* character value is reserved: “0” (Unicode character Zero, Unicode numeric value 30).
>
> This is used for the *DataOrigin* of `ShardKey.Empty` and `ShardChild.Empty`. Creating a [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) with a “zero” *DataOrigin* character but non-default (i.e. not zero or not null) *ShardId* or *RecordId* values will throw an [InvalidShardArgumentsException](/api/ArgentSea.InvalidShardArgumentsException.html) error.

This capability is useful for helping prevent data from being accessed with the wrong type of key — like an inventory key inadvertently passed to fetch an account record. Also, this may be helpful for caching data, since you can use the same dictionary to cache objects of different types without key collision.

> [!CAUTION]
> Although the *DataOrigin* is a char, it is serialized as an 8-bit ANSI charactor value. You should avoid using non-alphanumeric charactors for this value. And definately no emojis.

### The ShardId

The ShardId is used to identify a particular shard in the ShardSet. The core ArgentSea framework uses a generic type for the ShardId because the ideal data type will depend upon your requirements. Technically, the ShardId can be any of the types available to a RecordId (see below). Practically, however, it makes sense to avoid types without a corresponding SQL type and also avoid unnecessarily large data sizes. This leaves *byte* (SQL Server only), *short*, *char* as the most storage-efficient choices; *int*, *string* are viable choices if your ShardId has other requirements — like needing to integrate with external systems.

In essence, the most efficient ShardId type for SQL Server is byte/Tinyint, and for PostgreSQL is Int16(short)/Smalllint.

If you really can’t decide and have no particular requirements, a simple starting place is to use *byte* if are using SQL Server and you have confidence that you will never need more than 256 shards in a [ShardSet](/api/ArgentSea.ShardSetsBase-2.ShardSet.html), otherwise start with *short*.

Because the ShardId value is used in configuration, queries, and also for saving foreign shard references in your databases, once your project is established this value cannot be easily changed. The same ShardId type is used across all [ShardSets](/api/ArgentSea.ShardSetsBase-2.html).

> [!NOTE]
> The database itself may not know what its own *ShardId* is. This sounds absurd until you realize that it is genuinely difficult to keep scores or even hundreds of database schemas and procedures in sync while preserving a programmatic ShardId value. Your continuous delivery tooling will keep detecting any differences and trying to overwrite them! Fortunately, your connection *does* know this and can set the [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) values correctly.

### The RecordId

Like the ShardId, the RecordId is also an generic type, which can be one of the following:

| RecordId (and ChildId) Possible Data Types |
|--- |
| *byte*, *char*, *DateTime*, *DateTimeOffset*, *decimal*, *double*, *float*, *Guid*, *int*, *long*, *sbyte*, *short*, *string*, *TimeSpan*, *uint*, *ulong*, *ushort* |

If you have a data key that is not one of these types, the [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) objects will not know how to serialize the values.

Unlike the ShardId, the data type of the RecordId (and/or ChildId) need not be universal; it *can* be different for each table.

### The ChildId

The [ShardChild](/api/ArgentSea.ShardChild-3.html) type gets its name from the parent-child relationship that is typical of a two-column compound key. The [ShardChild](/api/ArgentSea.ShardChild-3.html) includes the *RecordId* of the [ShardKey](/api/ArgentSea.ShardKey-2.html) along with a new generic *ChildId* value. A *ShardGrandChild* could also be created to support three-level compound record keys, but, so far, there hasn’t been demand for that.

The ChildId can be any of the types listed in the previous section and the data type can also vary from table to table.

## Using The [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html)

Having a single object represent a compound record key adds only a little convenience. The real value comes from three capabilities: The shard Mapping attributes and the External key string.

### ToString(), ToExternalString(), and FromExternalString()

Calling `ToString()` on a [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) returns a list of the constituent values.

The `ToExternalKey()` function serializes the [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) values into a URL-safe string. This string also has a small amount of tampering protection. This is also the value returned when the model is serialized (i.e. in JSON results).

As you would expect, the `FromExternalString()` function reverses the operation, returning a ShardKey or ShardChild instance from a valid string.

The *External String* value can be used with, say, REST endpoints to specify a sharded record using a single argument.

## The [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) and [MapShardChild](/api/ArgentSea.MapShardChildAttribute.html) Attributes

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

This example sets the property to a [ShardKey](/api/ArgentSea.ShardKey-2.html) instance with a *DataOrigin* of “c”, the *ShardId* to the value of the data connection, and the *RecordId* the “CustomerId” column or parameter value.

The [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) attribute’s first argument is a *DataOrigin* char value. The second argument is the name of the data parameter or column. This name must *exactly* match the name in the data *MapTo* attribute. The [MapShardChild](/api/ArgentSea.MapShardChildAttribute.html) attribute is nearly identical, except for the additional *ChildId* parameter:

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

## Null Values

Because both [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) are structs, a variable or property of this type *cannot* be null. [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) objects are initialized to ShardKey.Empty or ShardChild.Empty respectively.

If a [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) represents a database field that might be Null, the [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) property or variable should be wrapped in the `Nullable<>` type. The *MapTo* attribute will set the `Nullable<ShardKey<>>` or `Nullable<ShardChild<>>` property to null if any of the constituent database column values are Null. If the underlying type is not `Nullable<>` and the database value is Null, the Mapper with throw an error (except as described in the next paragraph).

In most cases, a [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) represents a primary key, so a database Null value really represents a non-existent record. In this case, the desired behavior is probably to return the entire parent object as null. Marking the *MapTo* attribute(s) as *required* implements this behavior. When the *required* parameter is set, the [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) property does not need to be `Nullable<>` since a Null database value will return a null result object.

## Keyed Models

If your Model class uses a ShardKey or ShardChild key, you might consider implementing `IKeyedModel<,>` or `IKeyedChildModel<, ,>` respectively. These interfaces require a property named “Key” of ShardKey or ShardChild type. Models that implement this interface can leverage some additional ArgentSea utility functions.

For example, both the ShardKey and ShardChild structs have a static `Merge` method which can combine model sets, comparing the records by the key value. Both structs also have a `ForeignShards` method which returns a list of shards that are not local to the specified shard, which simplifies the problem of identifying records that need to be updated of foreign shards. Also, the SQL Server implementation also allows convertion of the Model keys directly to a Table Valued Parameter.

Next: [ShardSets](shardsets.md)
