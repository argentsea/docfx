# Query Batches

A *Batch* allows multiple commands to run within a single transaction on the same connection.  For example, ArgentSea makes it easy to save a single “customer” record with multiple “locations”. However, using a batch you could save multiple customers, each with multiple locations — all within the same transaction.

The real purpose of the *Batch* is that it allows non-query actions, like *SqlBulkCopy* (SQL Server) or the *NpgsqlBinaryImporter* (PostgreSQL) to be combined with queries within the same transaction. See [multi-record saves](multirecord.md) for more information.

> [!CAUTION]
> Because they involve multiple round-trips to the database server, Batches are less efficient than executing multiple SQL statements in a single command. You should avoid using batches to execute a series of statements that could be combined into a single command.

> [!NOTE]
> Because client-managed transactions are much less efficient than server-side transactions, a *Batch* is the only place where ArgentSEa explicitly enlists ADO.NET transactions.

## Batch Types

There are three types of batches. Each type offers somewhat different operations.

* __DatabaseBatch__ can be used for non-sharded database connections.
* __ShardBatch__ is for a specific shard in a shard set.
* __ShardSetBatch__ can run the batch commands on every shard in the shard set. These commands cannot return a result.

Batches are executed with the `RunAsync` command. The ShardSet’s `RunAsync` method will only accept a `ShardSetBatch` argument. Likewise, the Database or Shard connections will only accept the `DatabaseBatch` or `ShardBatch` respectively.

Batches are simply collections of *BatchStep* objects. The BatchStep is abstract. You can implement your own BatchStep, but several implementations are available. The principal one enables execution of `Query`.

ShardSet batches cannot not return a result, but the other batches use a generic argument to specify the type of the return value. For example, this batch will return a integer value when run:

```csharp
var batch = new DatabaseBatch<int>();
```

You use the `Add` method to set up the batch commands. The `Add` method has a fluent API:

```csharp
var batch = new DatabaseBatch<string>()
    .Add(Queries.CustomerLoadStuff, parameters)
    .Add(Queries.CustomerCreate, "customername");
var customerName = _database.Write.RunAsync(batch, cancellation);
```

In this example, the batch is declared with a return type (of integer). The second step run a query that ultimately returns a value; the value in the first row with a column name of “customername” is returned to the caller.

### The Shard Set Batch

Because the `ShardSetBatch` will run on multiple sharded databases, the `ShardSetBatch` does return data when executed. ArgentSea offers the ability to run a query (`QueryProcedure` or `QueryStatement`) on a `ShardSetBatch`, although a user-created implementation of a custom BatchStep&lt;&gt; could also be used.

When creating a `ShardSetBatch`, the generic parameter is the shard id type.

```csharp
var batch = new ShardSetBatch<short>()
    .Add(Queries.UpdateCustomers, parameters)
    .Add(Queries.ProcessCustomers);
await _shardSet.Write.RunAsync(batch, cancellation);
```

The query can include input parameters. As with other query commands, if a *shardParameters* argument is provided, only the listed shards with be impacted — and the shard parameter values will be updated as per any matching argument values.

| Instantiation: | Batch argument: | Return Type: |
| --- | --- |--- |
| new ShardSetBatch&lt;TShard&gt;() | Query | none |
| Custom BatchStep implementation | varies | none |


> [!CAUTION]
> The `ShardSetBatch` will likely perform more poorly than a single SQL command. The main circumstance where this would be useful is when the client application must dynamically assemble the Query set. Really, the `ShardSetBatch` seems to have a fairly limited number of use cases.

### The Shard Batch

The `ShardBatch` can execute a query and return a Model result (using the Mapper), a ShardKey or a list of ShardKeys. For example, a batch that inserts records with identity columns might need to return the ShardKey(s) containing the identifiers of the newly inserted records. You can use `ShardKey` for a single record key result, or `List<ShardKey>` or `IList<ShardKey>` for a multi-record key result.

The `ShardBatch` has two generic parameters. The first is the ubiquitous shard id type; the second is the return type.  

> [!NOTE]
> The methods that are available are determined by the return type specified in the second generic argument.

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

> [!NOTE]
> If you do not need a result, you can simply specified the return type of `object` and the Run method will return null.

Ideally, only one step in the batch should return a result. If multiple steps each return a result, only the last one with a valid value (non-null or non-default value) is returned to the client.

| Instantiation: | Batch argument: | Return Type: |
| --- | --- |--- |
| new ShardBatch&lt;TShard, ShardKey&lt;TShard, TRecord&gt;&gt;() | Query | ShardKey |
| new ShardBatch&lt;TShard, ShardKey&lt;TShard, TRecord, TChild&gt;&gt;() | Query | ShardKey |
| new ShardBatch&lt;TShard, List&lt;ShardKey&lt;TShard, TRecord&gt;&gt;&gt;() | Query | ShardKey List |
| new ShardBatch&lt;TShard, List&lt;ShardKey&lt;TShard, TRecord, TChild&gt;&gt;&gt;() | Query | ShardKey List |
| new ShardBatch&lt;TShard, IList&lt;ShardKey&lt;TShard, TRecord&gt;&gt;&gt;() | Query | ShardKey List |
| new ShardBatch&lt;TShard, IList&lt;ShardKey&lt;TShard, TRecord, TChild&gt;&gt;&gt;() | Query | ShardKey List |
| new ShardBatch&lt;TShard, Model&gt;() | Query | Model |
| new ShardBatch&lt;TShard, CustomBatchStep) | varies | any |

### The Database Batch

The DatabaseBatch object is similar to the ShardBatch object. It has one generic argument, which specifies the return type when executed.

```csharp
var batch = new DatabaseBatch<long>()
    .Add(Queries.CustomerCreate, parameters)
    .Add(Queries.CustomerList, "customerid");
var newCustomerId = _database.Write.RunAsync(batch4, cancellation);

```

The return type can be a Model class (using the Mapper), a column value, or a list of column values. This allows you to return the identity value (or values) of an inserted record (or records).

> [!NOTE]
> The methods that are available are determined by the return type specified in the generic argument.

For example, this would return the identity value of the new customer:

```csharp
var batch = new DatabaseBatch<int>()
    .Add(Queries.CustomerPrep)
    .Add(Queries.CustomerCreate, parameters, "customerid");
var customerId = await _database.Write.RunAsync(batch, cancellation);
```

To instead return a list of values from the query, make the return type a list:

```csharp
var batch = new DatabaseBatch<List<int>>()
    .Add(Queries.CustomerPrep)
    .Add(Queries.CustomersCreate, parameters, "customerid");
var customerIds = await _database.Write.RunAsync(batch, cancellation);
```

> [!NOTE]
> If you do not need a result, you can simply specified the return type of `object` and the Run method will return null.

Ideally, only one step in the batch should return a result. If multiple steps each return a result, only the last one with a valid value (non-null or non-default value) is returned to the client.

| Instantiation: | Batch argument: | Return Type: |
| --- | --- |--- |
| new DatabaseBatch&lt;TRecord&gt;() | Query | TRecord |
| new DatabaseBatch&lt;int&gt;() | Query | int |
| new DatabaseBatch&lt;List&lt;TRecord&gt;&gt;() | Query | Id List |
| new DatabaseBatch&lt;IList&lt;TRecord&gt;&gt;() | Query | Id List |
| new DatabaseBatch&lt;Model&gt;() | Query | Model |
| new DatabaseBatch&lt;CustomBatchStep) | varies | any |

Next: [Multi-record Saves](multirecord.md)
