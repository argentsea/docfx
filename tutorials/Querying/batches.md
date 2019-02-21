# Query Batches

Batches allows multiple commands to run on the same connection within a single transaction. Because they involve multiple round-trips to the database server, Batches are less efficient than executing multiple SQL statements in a single command. You should avoid using batches to execute a series of statements that could be combined into a single command, but you may occasionally need access to a single connection for multiple operations. For example, you might want to use the connection to batch insert a series of records, then run a query to validate and process the data. In fact, support for PostgreSQL’s COPY functionality is why batches were created.

> [!NOTE]
> Because client-managed transactions are much less efficient than server-side transactions, only ArgentSea batches enlist ADO.NET transactions.

## Batch Types

There are three types of batches, each offering somewhat different operations.

* __DatabaseBatch__ can be used for non-sharded database connections.
* __ShardBatch__ is for a specific shard in a shard set.
* __ShardSetBatch__ can run the batch commands on every shard in the shard set. These commands cannot return a result.

Batches are executed with the `RunAsync` command. The ShardSet’s `RunAsync` method will only accept a `ShardSetBatch` argument. Likewise, the Database or Shard connections will only accept the `DatabaseBatch` or `ShardBatch` respectively.

Batches are simply collections of (abstract) BatchStep objects. You use the `Add` method to set up the batch commands. The `Add` method has a fluent API:

```csharp
var batch = new DatabaseBatch<int>()
    .Add(Queries.CustomerLoadStuff, parameters)
    .Add(Queries.CustomerCreate, "customerid");
var newCustomerId = _database.Write.RunAsync(batch, cancellation);
```

In this example, the batch is declared with a return type (of integer). The second step run a query that ultimately returns a value. The value with a column name of “customerid” in the first row is returned to the caller.

### The Shard Set Batch

The `ShardSetBatch` does not return any data. The ShardSetBatch only supports running a query (or a user-supplied implementation of a custom BatchStep&lt;&gt;).

```csharp
var batch = new ShardSetBatch<short>()
    .Add(Queries.UpdateCustomers)
    .Add(Queries.ProcessCustomers);
await _shardSet.Write.RunAsync(batch, cancellation);
```

The generic parameter is the (ubiquitous) shard id type.

The query can include input parameters. If a *shardParameters* argument is provided, only the listed shards with be impacted — and the shard parameter values will be updated as per any matching argument values.

| Instantiation: | Batch argument: | Return Type: |
| --- | --- |--- |
| new ShardSetBatch&lt;TShard&gt;() | Query | none |
| Custom BatchStep implementation | varies | none |

### The Shard Batch

The `ShardBatch` can execute a query and return a ShardKey, a ShardChild, a list of ShardKeys or ShardChilds, or Model object. The first generic parameter is the ubiquitous shard id type, you determine the return type when you instantiate the ShardBatch. 

> [!NOTE]
> The second generic specifies both the return type and also what methods are available. You can use `ShardKey`, `ShardChild` for a single record key result, or `List<ShardKey>`, `List<ShardChild>`, `IList<ShardKey>` and `IList<ShardChild>` for a multi-record key result.

For example, this would return the ShardKey of the new customer:

```csharp
var batch = new ShardBatch<short, ShardKey<short, int>>()
    .Add(Queries.CustomerCreate)
    .Add(Queries.CustomerGet, parameters, 'c', "customerid");
var customerKey = await _shardSet.DefaultShard.Write.RunAsync(batch, cancellation);
```

Note that you must specify the *DataOrigin* value for the new ShardKey and the column name from which to get the new record id.

To instead return a list of new keys from the query, make the return type a list:

```csharp
var batch = new ShardBatch<short, List<ShardKey<short, int>>>()
    .Add(Queries.CustomersCreate)
    .Add(Queries.CustomersGet, parameters, 'c', "customerid");
var customerKeys = await _shardSet.DefaultShard.Write.RunAsync(batch, cancellation);
```

In both the above examples, the shard id of the resulting key will be set to the shard id of the current shard. If the query result contains shardkeys that reference other shards, simply provide the ShardId column name also:

```csharp
var batch = new ShardBatch<short, List<ShardKey<short, int>>>()
    .Add(Queries.CustomersCreate)
    .Add(Queries.CustomersGet, parameters, 'c', "shardid", "customerid");
var customerKeys = await _shardSet.DefaultShard.Write.RunAsync(batch, cancellation);
```

Ideally, only one step should return a result. If multiple steps each return a result, only the last one with a valid value (non-null or non-default value) is returned to the client.

| Instantiation: | Batch argument: | Return Type: |
| --- | --- |--- |
| new ShardBatch&lt;TShard, ShardKey&lt;TShard, TRecord&gt;&gt;() | Query | ShardKey |
| new ShardBatch&lt;TShard, ShardChild&lt;TShard, TRecord, TChild&gt;&gt;() | Query | ShardChild |
| new ShardBatch&lt;TShard, List&lt;ShardKey&lt;TShard, TRecord&gt;&gt;&gt;() | Query | ShardKey List |
| new ShardBatch&lt;TShard, List&lt;ShardChild&lt;TShard, TRecord, TChild&gt;&gt;&gt;() | Query | ShardChild List |
| new ShardBatch&lt;TShard, IList&lt;ShardKey&lt;TShard, TRecord&gt;&gt;&gt;() | Query | ShardKey List |
| new ShardBatch&lt;TShard, IList&lt;ShardChild&lt;TShard, TRecord, TChild&gt;&gt;&gt;() | Query | ShardChild List |
| new ShardBatch&lt;TShard, Model&gt;() | Query | Model |
| new ShardBatch&lt;TShard, CustomBatchStep) | varies | any |

### The Database Batch

The DatabaseBatch object is similar to the ShardBatch object, except the return values avoid the ShardKey or ShardChild values.

| Instantiation: | Batch argument: | Return Type: |
| --- | --- |--- |
| new DatabaseBatch&lt;TRecord&gt;() | Query | TRecord |
| new DatabaseBatch&lt;int&gt;() | Query | int |
| new DatabaseBatch&lt;List&lt;TRecord&gt;&gt;() | Query | Id List |
| new DatabaseBatch&lt;IList&lt;TRecord&gt;&gt;() | Query | Id List |
| new DatabaseBatch&lt;Model&gt;() | Query | Model |
| new DatabaseBatch&lt;CustomBatchStep) | varies | any |


## PostgreSQL