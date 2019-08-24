# The [ShardKey](/api/ArgentSea.ShardKey-2.html)

All databases need a way to uniquely identify a record — a record key. With sharded data sets, a record key need to be unique across all the shards. Within a single database, uniqueness is easily managed; across a shard set, database engines can no longer enforce uniqueness for data they don’t know about. Additionally, on the client side, the query dispatcher needs to be able to use the record key in order to know to which shard connection to use.

There are two approaches to maintaining a unique key across multiple databases:

* __Use distinct identity ranges for each database in the shard set.__ The upside of this approach is that it is possible to combine data sets without conflicts; the downside is that configuration is complicated — on both the client and database servers — so mistakes are more likely, and some mistakes can be *very* hard to fix. The query dispatcher must know the various identity ranges hosted by each server in order to select the right connection.
* __Combine the *shard connection key* and the *record key* into a larger “compound key”.__ With this approach, finding the right shard connection is easy because the value is embedded in the compound record key. The database servers do not need to be configured with separate identity ranges, which in some case may allow smaller, more efficient key sizes (i.e. int vs bigint). Combining or splitting shards could be more complicated, however.

ArgentSea will work with either design. The [ShardKey](/api/ArgentSea.ShardKey-1.html) object offers support for the second approach.

## Components

A [ShardKey](/api/ArgentSea.ShardKey-1.html) consists of three essential components: a *DataOrigin* char value, a *ShardId*, and a *RecordId*. ShardKey instances with additional generic types can be used for compound table keys; these will had the additional *ChildId*, *GrandChildId*, and *GreatGrandChildId* values.

<table border="0" margin="0" padding="0"><tr><td width="43%"><img src="/images/shardkey.svg"></td><td width="57%"><img src="/images/shardchild.svg"></td></tr></table>

### The DataOrigin

[ShardKey](/api/ArgentSea.ShardKey-1.html) types have a *DataOrigin* value. The purpose of this value is to represent a distinct data source (table). It is simply a character value that you can choose to differentiate them.

For example, keys representing a *Customer* record might have a *DataOrigin* of “c”, whereas keys representing a *Product* record might have a *DataOrigin* of “p”. Because this simple tag identifies the data source, two different [ShardKeys](/api/ArgentSea.ShardKey-1.html) from the same shard and with the same record number will still not be equal because they represent different source data.

> [!IMPORTANT]
> One *DataOrigin* character value is reserved: “0” (Unicode character Zero, Unicode numeric value 30).
>
> This is used for the *DataOrigin* of `ShardKey.Empty`. Creating a [ShardKey](/api/ArgentSea.ShardKey-1.html) with a “zero” *DataOrigin* character but non-default (i.e. not zero or not null) *ShardId* or *RecordId* values will throw an [InvalidShardArgumentsException](/api/ArgentSea.InvalidShardArgumentsException.html) error.

This capability is useful for helping prevent data from being accessed with the wrong type of key — like an inventory key inadvertently passed to fetch an account record. Also, this may be helpful for caching data, since you can use the same dictionary to cache objects of different types without key collision.

> [!CAUTION]
> Although the *DataOrigin* is a char, it is serialized as an 8-bit ANSI character value. You should avoid using non-alphanumeric characters for this value. And definitely no emojis.

### The ShardId

The ShardId is used to identify a particular shard in the ShardSet. This is a 16-bit integer, also called Int16, short, or Smallint in SQL. Because it only requires two bytes, it is very compact in storage and memory, yet it can theoretically support over 65,000 shards.

> [!NOTE]
> The earliest versions of the ArgentSea framework used a generic type for the ShardId — similar to that used for the RecordId, ChildId, etc. — which allowed the shard identifier to be a string, byte, Guid, etc. However, this created a verbose object model where the consuming code had to endlessly declare the ShardId type. Within a given project this type could never change, so this created a lot of redundancy.
> Given the types available in both .Net and SQL, the efficiency of data storage, and the likely maximum number of shards, one type was clearly optimal: ArgentSea was rewritten so that the ShardId type was always a 16-bit integer.

> [!CAUTION]
> The database shards themselves may not know what their own *ShardId* is! The ShardId comes only from the connection information in the local configuration file. This means that a misconfiguration can corrupt data! For example, if a user attempts to update the record with Id 2 on shard 4 but a misconfiguration directs the connection to the wrong shard, the command will overwrite the record with id 4 — in the wrong database!
> Consider using optimistic concurrency practices like a rowversion or timestamp. This validation will also protect against updating the wrong database.

### The RecordId

Like the ShardId, the RecordId is also an generic type, which can be one of the following:

| RecordId (and ChildId/GrandChildId/GreatGrandChildId) Possible Data Types |
|--- |
| *byte*, *char*, *DateTime*, *DateTimeOffset*, *decimal*, *double*, *float*, *Guid*, *int*, *long*, *sbyte*, *short*, *string*, *TimeSpan*, *uint*, *ulong*, *ushort* |

If you have a data key that is not one of these types, the [ShardKey](/api/ArgentSea.ShardKey-1.html) class will not know how to serialize the values.

### The ChildId, GrandChildId, and GreatGrandChildId

The [ShardKey](/api/ArgentSea.ShardKey-1.html) type gets its name from the parent-child relationship that is typical of a two-column compound key. By specifying additional generic arguments, you can extend the object to accommodate a compound key of up to four columns!

## Using The [ShardKey](/api/ArgentSea.ShardKey-1.html)

Having a single object represent a compound record key adds only a little convenience. The real value comes from three capabilities: The shard Mapping attributes and the External key string.

### ToString(), ToExternalString(), and FromExternalString()

Calling `ToString()` on a [ShardKey](/api/ArgentSea.ShardKey-1.html) returns a json-formatted list of the constituent values.

The `ToExternalKey()` function serializes the [ShardKey](/api/ArgentSea.ShardKey-1.html) values into a URL-safe string. This string also has a small amount of tampering protection. This is also the value returned when the model is serialized (i.e. in JSON results).

As you would expect, the `FromExternalString()` function reverses the operation, returning a ShardKey instance from a valid input string.

The *External String* value can be used with, say, REST endpoints to specify a sharded record using a single argument.

## The [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) Attribute

The [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) and attribute maps the shard information, record key, and (as appropriate) the child record values to a new [ShardKey](/api/ArgentSea.ShardKey-1.html) instance. The simplest implementation is to simply add the [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) and the type-appropriate *MapTo* attribute(s).

## [SQL Server](#tab/tabid-sql)

````csharp
[MapShardKey('c', "@CustomerId")]
[MapToSqlInt("@CustomerId")]
public ShardKey<byte, int> CustomerKey { get; set; }

````

## [PostgreSQL](#tab/tabid-pg)

````csharp
[MapShardKey('c', "customer_id")]
[MapToPgInteger("customer_id")]
public ShardKey<short, int> CustomerKey { get; set; }
````

***

This example sets the property to a [ShardKey](/api/ArgentSea.ShardKey-1.html) instance with a *DataOrigin* of “c”, the *ShardId* to the value of the data connection, and the *RecordId* the “CustomerId” column or parameter value.

The [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) attribute’s first argument is a *DataOrigin* char value. The second argument is the name of the data parameter or column. This name must *exactly* match the name in the data *MapTo* attribute. If [ShardKey](/api/ArgentSea.ShardKey-1.html) has child columns, include those column definition too (ensuring that these names also exactly match).

## [SQL Server](#tab/tabid-sql)

````csharp
[MapShardKey('O', "@OrderId", "@OrderItemId")]
[MapToSqlBigInt("@OrderId")]
[MapToSqlSmallInt("@OrderItemId")]
public ShardKey<byte, long, short> OrderItemKey { get; set; }

````

## [PostgreSQL](#tab/tabid-pg)

````csharp
[MapShardKey('O', "order_id", "order_item_id")]
[MapToPgBigint("order_id")]
[MapToPgInteger("order_item_id")]
public ShardKey<short, long, int> OrderItemKey { get; set; }
````

***

In both previous examples, the ShardId will be *implicitly* obtained from the *connection’s* ShardId. In the case of results that include the primary key column, this works well. However, when a data record references a table belonging to a foreign shard, the *ShardId* must be *explicitly* set by the database record. To do this, just add a *ShardID* column name to the first argument of the *MapShardKey* attribute:

## [SQL Server](#tab/tabid-sql)

````csharp
[MapShardKey("@CustomerShardId", 'c', "@CustomerId")]
[MapToSqlInt("@CustomerId")]
public ShardKey<byte, int> CustomerKey { get; set; }

[MapShardKey("@OrderShardId", 'O', "@OrderId", "@OrderItemId")]
[MapToSqlBigInt("@OrderId")]
[MapToSqlSmallInt("@OrderItemId")]
public ShardKey<long, short> OrderItemKey { get; set; }

````

## [PostgreSQL](#tab/tabid-pg)

````csharp
[MapShardKey("customer_shard_id", 'c', "customer_id")]
[MapToPgInteger("customer_id")]
public ShardKey<short, int> CustomerKey { get; set; }

[MapShardKey("order_shard_id", , 'O', "order_id", "order_item_id")]
[MapToPgBigint("order_id")]
[MapToPgSmallint("order_item_id")]
public ShardKey<long, short> OrderItemKey { get; set; }
````

***

## Null Values

Because a [ShardKey](/api/ArgentSea.ShardKey-1.html) is a  struct, a variable or property of this type *cannot* be null. [ShardKey](/api/ArgentSea.ShardKey-1.html) objects are initialized to ShardKey.Empty.

If a [ShardKey](/api/ArgentSea.ShardKey-1.html) represents a database field that might be Null, the [ShardKey](/api/ArgentSea.ShardKey-1.html) property or variable should be wrapped in the `Nullable<>` type. The *MapToShardKey* attribute will set `Nullable<ShardKey<>>` or `ShardKey<>?` properties to null if any of the constituent database column values are Null. If the underlying type is not `Nullable<>` and the database value is Null, the Mapper with throw an error (except as described in the next paragraph).

In many cases, a [ShardKey](/api/ArgentSea.ShardKey-1.html) will represent a primary key, so a database Null value really represents a non-existent record. In this case, the desired behavior is probably to return the entire parent object as null. Marking the *MapToShardKey* attribute(s) as *required* implements this behavior. When the *required* parameter is set, the [ShardKey](/api/ArgentSea.ShardKey-1.html) property does not need to be `Nullable<>` since a Null database value will return a null result object.

## Keyed Models

If your Model class uses a ShardKey key, you might consider implementing `IKeyedModel<,>` or `IKeyedChildModel<, ,>` respectively. These interfaces require a property named “Key” of ShardKey type. 

Models that implement this interface can leverage some additional ArgentSea utility functions. For example, the ShardKey struct has a static `Merge` method which can combine model sets, comparing the records by the key value. Both structs also have a `ForeignShards` method which returns a list of shards that are not local to the specified shard, which simplifies the problem of identifying records that need to be updated of foreign shards. 

Also, the SQL Server implementation also allows conversion of the Model keys directly to a Table Valued Parameter. Likewise, the PostgreSQL implementation can use model keys to write values into a temporary table.

Next: [ShardSets](shardsets.md)
