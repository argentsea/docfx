# Fetching Data

Retrieving database data consists of running a query on each connection. ArgentSea provides various methods to offer the best approach.

## Connection methods

Both database connections and shards have distinct Read and Write connections. The distinction allows the system to “scale out” reads and writes. The Read connection should be used for SELECT-only stored procedures or SQL statements; the Write connection should be used for everything else. For example, you could direct reads to a mirror, active standby, or read-only endpoint, and direct writes to the master or source database.

> [!TIP]
> Even if you do not *currently* have separate read-only endpoints like mirrors or active standbys, consistent discrimination of Read and Write access will allow you to scale-out in the future.

If only the Read or Write connection is configured, both the Read and Write connections will have the same value.

These are the methods that can be invoked on a connection:

| Method | Description |
| --- | --- |
| __ReturnValueAsync__ | Returns a value from the database. This may be a return value (int) or single output value from a parameter or column. |
| __RunAsync__ | Executes a database statement, procedure, or batch. No results are returned (except possibly batch results). |
| __QueryAsync__ | Returns the typed object created by a handler delegate. |
| __MapListAsync__ | Returns a List of typed objects from the data results. |
| __MapReaderAsync__ | Returns a typed object created by the [Mapper](/api/ArgentSea.Mapper.html) from __DataReader__ results. |
| __MapOutputAsync__  | Returns a typed object created by the [Mapper](/api/ArgentSea.Mapper.html) from output parameters *and* __DataReader__ results. |

Database objects and shard instances have both `Write` and `Read` connections, which executes a query on a single database. All the methods listed are available on either connection (even though it may not make sense to use `RunAsync` on a read connection).

The ShardSet has `Write`, `ReadAll`, and `ReadFirst` connections, which execute the procedure or SQL statement on every shard. They return either the combined result or the first valid (non-null) result. For example, if you need to look up a user by their login name (rather than their user key), use `ReadFirst` to query all shards for a matching record, and to return the single expected matching result. Whereas, `ReadAll` could be used to retrieve all users in any shard with particular attribute. Note that `ReadAll` methods always return list results.

| Method | Uses Mapper | Db.Read | Db.Write | ShardSet.ReadAll | ShardSet.ReadFirst | ShardSet.Write |
| --- |:---:| :---: | :---: | :---: | :---: | :---: |
| __ReturnValueAsync__ |   | • | • |  |  |  |
| __RunAsync__ |   | • | • |  |  | • |
| __QueryAsync__ |   | • | • | • | • | • |
| __ListAsync__ |   | • | • | • | • | • |
| __MapListAsync__ | • | • | • | • |  |  |
| __MapReaderAsync__ | • | • | • | • | • | • |
| __MapOutputAsync__ | • | • | • | • | • | • |

## Method Arguments

The arguments are largely consistent across all of the methods, except running a query batch which is discussed later.

### Required Argument: (Query) query

There are two types of query objects:

* __Stored procedures__ are parameterized statements stored within the database server. This is the preferred approach with SQL Server.
* __Statements__ are parameterized SQL files stored in a application folder (rather than compiled into the source code). This is useful for PostgreSQL databases and situations with less-than-full ownership of the target database.

The recommended practice is to create a single static class rendering all SQL statements and procedures. This approach is simple and also provides a reference count indicating which procedures/statements are in use, which can be a problem when an application has grown large. Sample code and more detail is discussed in [SQL queries](/tutorials/querying/sql.md) and the [configuration quickstart](/tutorials/quickstarts/configuration.md).

An example invocation is like this:

```csharp
await database.RunAsync(Queries.MyProcedure, parameters, cancellationToken);
```

### Required Argument: (DbParameterCollection) parameters

The abstract `DbParameterCollection` is implemented by ArgentSea’s [QueryParameterCollection](/api/ArgentSea.QueryParameterCollection.html) object. Because it is also implemented by the provider-specific command.Parameters property, if you have a command with valid parameters defined (for some reason), you can use that too.

This value can be null if there are no parameters.

> [!WARNING]
> When working with output parameters in standard ADO.NET, you may habitually maintain a variable reference to any output parameters you created before adding it to the collection. This makes it easy to get the output parameter value after the query is executed.  
> This approach will not work with sharded data, because ArgentSea will *copy* the parameter set before executing the query. Any referenced output parameters will *not* contain a data result.

### Optional ShardSet Argument: (IEnumerable&lt;ShardParameterValue&lt;TShard&gt;&gt;) shardParameterValues

Some shard query method overloads accept a `ShardParameterValue` object. This object allows you to specify which shards should be queried and even provide distinct parameter values to each shard.

For example, suppose your User record returns a list of “Friends”. The Friend detail data may be hosted on other shards, but not on *every* shard. Building a list of `ShardParameterValue` objects from the User results would limit the subsequent queries to just the relevant shards.

The `ShardParameterValue` type has a ShardId and an optional parameter name and value. Only shards with at least one listed ShardId will be queried. If a parameter name is also specified, the corresponding parameter will be set to that value on the indicated shard. You can include multiple parameters/values on the same shard by repeating the shardId.

### Optional ShardSet Argument: (string) shardParameterName

Some shard query overloads also accept the name of the parameter that represents the name of the parameter that should be set to shardId value. If specified, ArgentSea will set this parameter value to the current shardId value as it executes each query.

For example, a query for a list of records that spans shards could be enhanced if the query new the value of its own ShardId. Alternatively, because a shard misconfiguration might result in catastrophic data corruption (due to the high likelihood of duplicate record identities between shards), you might require that queries that write to the database also have a ShardId parameter that they validate is correct.

### Optional Argument: (QueryResultModelHandler&lt;TShard, TArg, TModel&gt;>) resultHandler

This is only used in the QueryAsync methods. As described earlier, the data query process is divided into two processes. The resultHandler is a delegate that may be invoked concurrently by distinct, shard-specific threads.

If you use a data access method prefixed with Map*, this argument is not required because the delegate provided by the Mapper is used. If the Mapper does not suit your purpose, then a custom delegate must be provided to a Query* method.

Your custom delegate can have an argument that provides additional data or context information. Information on how to build a custom delegate is provided below.

### Optional Argument: (bool) isTopOne

Some overloads expose the isTopOne option, which allows a minor optimization when only a single result is expected. For example, if you are looking up a record by its key, you don’t need to allocate space for multiple results when only a single result can ever be returned.

### Optional Argument: (TArg) optionalArgument

If you are creating a custom data handling method, you may need to provide additional data or context information. This argument may be generically typed. The provided object is passed to your result handling delegate.

### Required Argument: (CancellationToken) cancellationToken

The cancellation token allow you to cancel asynchronous operations. ASP.NET MVC provides cancellation tokens and these can be passed along. In this way, when a user abandons their session, any uncompleted queries can be cancelled.

## The MapReader&ast; and MapOutput&ast; Methods

The __MapReader&ast;__ and __MapOutput&ast;__ methods are similar. Both use the Mapping attributes to resolve data to Model objects. The __MapOutput&ast;__ method uses *output parameters* to build the root result object; the __MapReader__ methods use a (single record) __DataReader__ result instead.

So, if you use output parameters (which is potentially more performant), use __MapOutput&ast;__. If you use standard SELECTs to return your data, use __MapReader&ast;__.

Both methods support multiple result sets that populate properties that contain Lists (`List<Model>` or `IList<Model>`) of related data. For example, you might have an Order record with a property containing an OrderItem List. The list items come from (additional) DataReader results. A single root Model may have up to eight of these List properties. The List property must be settable.

> [!NOTE]
> The order in which your attribute-mapped class appears in the generic definitions should be the same order as the list data results in the procedure or statement output.

An example of calling each would be:

```csharp
// In this example, ws.GetOrderDetails returns Order data in output parameters:
_database.MapOutputAsync<Order>(Queries.GetOrderDetails, parameters, cancellation);
// Here, ws.GetOrderDetails returns simple Order data in a single-row SELECT:
_database.MapReaderAsync<Order>(Queries.GetOrderDetails, parameters, cancellation);

// Now ws.GetOrderDetails returns Order data in output parameters and a list of OrderItem from a SELECT:
_database.MapOutputAsync<Order, OrderItems>(Queries.GetOrderDetails, parameters, cancellation);
// Finally, ws.GetOrderDetails returns Order data in a single-row SELECT, then a list of OrderItems from a 2nd SELECT:
_database.MapReaderAsync<Order, Order, OrderItems>(Queries.GetOrderDetails, parameters, cancellation);

// Expanding this, we now have output parameters and three SELECTs:
_database.MapOutputAsync<Store, OrderHistory, Locations, Contact>(Queries.GetStoreDetails, parameters, cancellation);
// Likewise, the query now returns four SELECTs, and the third one is a single-row SELECT with the base customer data,
// the remaining select are used to build customer property lists (order history, locations, and contacts):
_database.MapReaderAsync<Store, OrderHistory, Store, Locations, Contact>(Queries.GetStoreDetails, parameters, cancellation);
```

In both methods, the generic type in the *first* position is the return type. If additional results are included in the result stream, the subsequent types define the order in which they are expected in the DataReader results. You can have up to eight DataReader results streamed to distinct List properties.

* In the __MapOutput&ast;__ example, then, the result type is Order and the first DataReader result is a series of OrderItems.
* In the __MapOutput&ast;__ example, the result type is Order, and the first DataReader result is the Order data, and the second DataReader result is a series of OrderItems.

## The Query&ast; Methods

The __Query&ast;__ methods provide the most control, as you are given raw ADO.NET query results to construct whatever return value you like. You can return a list, a dictionary, or any type of Model object. When you call a __Query&ast;__ method, you must provide a handler method whose signature corresponds to the [QueryResultModelHandler](/api/ArgentSea.QueryResultModelHandler-3.html) delegate.

There are two obvious scenarios for the __Query&ast;__ methods:

* The Model class is defined in a library, so Mapping attributes cannot be added.
* The rendering a complex return value is beyond the capabilities of the Mapper.

The delegate even has a parameter that allows you to provide custom data (through the query method) with which to construct your result object.

The delegate must be thread-safe.  The [ShardSet](/api/ArgentSea.ShardSetsBase-2.ShardSet.html) manages the complexity of initializing multiple queries on multiple connections and multiple results, but it is the delegate that takes the database results (from each connection/thread) and creates an object result.

> [!NOTE]
> The Mapper provides several thread-safe, high-performance [QueryResultModelHandler](/api/ArgentSea.QueryResultModelHandler-3.html) delegates. In fact, providing a Mapper delegate to the __Query&ast;__ method is exactly how the __MapOutput&ast;__ an __MapOutput&ast;__ methods are implemented. You can use this yourself to extend the Mapper; just provide your own delegate that calls the Mapper in turn.

Details on implementing the [QueryResultModelHandler](/api/ArgentSea.QueryResultModelHandler-3.html) delegate is in the next section.

Next: [Handling Data Results](handling.md)
