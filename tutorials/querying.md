# Querying Data

This deep dive assumes that you have your project correctly [configured](configuration.md) and it is also helpful if you have some understanding of ArgentSea’s [Mapping](mapping.md) capabilities

ArgentSea was originally built to support application data sharding. Even if you do not use data sharding in your application, a brief discussion of the issues will help explain the architecture behind of ArgentSea’s data access approach. It will not be more difficult than any other ADO.NET query.

## Understanding Sharding

The best way to understand the query architecture of ArgentSea is to describe a typical ADO.NET query then describe how this must change to account for concurrent multi-threaded queries across a shard set. These changes are not complicated, and they should be helpful even if your project does use sharding.

A typical ADO.NET data access method follows these steps:

1. Start with a __connection__ object, created from a connection string.
2. Create a __command__ object that is associated with the __connection__ object.
3. Next, the populate the command's *Parameters* property with the necessary input and output parameters.
4. Open the connection and run the command.
5. Create a Model object, which represents the data to the application, and use the __DataReader__ and/or output parameters to populate its properties.

In a sharded environment, however:

* The same parameters must be executed on multiple connections — reversing the steps 1 to 3.
* A distinct command object must be executed and the results processed on a separate thread for each connection. The parameters cannot be shared (different threads would overwrite each other’s values) and the result handler must be thread-safe because it could be simultaneously executing on different connections.

ArgentSea manages the challenges of multi-threaded access access with a four-step approach:

1. Declare the parameters and arguments that will be passed to the stored procedures.
2. Create a thread for each shard connection, then create the __connection__ (and __command__) object for each.
3. Copy the parameter values to the parameter collection on each shard’s __command__ object.
4. Call the stored procedure/function on each shard’s thread. When results are obtained, call (thread-safe) code to create and populate a Model object.
5. Merge the results and return them to the caller.

Ultimately, using ArgentSea on multiple shards is no more difficult than writing simple ADO.NET database access code (and usually much easier), but the code new needs to be grouped and sequenced differently. Previously, you would usually use just one data access procedure, which would set the ADO.NET parameters, run the query, then convert the results to a Model object. Now, because processing results is multi-threaded whereas setting up the query is not, you need to split that process into *two* procedures:

* The __*caller*__ method sets the parameters and calls an ArgentSea query method. This executes on a single thread.
* The __*handler*__ procedure converts the results (__DataReader__ and/or output parameters) to a Model object result. This can execute on many threads.

This ArgentSea query paradigm applies even to non-sharded queries using the Databases collection. This provides some design consistency, but also enables the Mapper for both sharded and non-sharded data.

> {!TIP]
> If you use ArgentSea’s optional [Mapping](mapping.md) functionality, the multi-threaded results handling procedure is *already provided* by the Mapper. You do not have to write a handler.

## Setting Parameters

With the ArgentSea framework, you need to set parameter values *before* a connection or command is created. The ADO.NET standard parameter collections cannot be created without a command object host. To fill this need, ArgentSea provides a [QueryParameterCollection](/api/ArgentSea.QueryParameterCollection.html) object, which is simply a collection of ADO.NET DbParameters. This object allow you to create an instance with a simple `new` statement.

```csharp
var parameters = new QueryParameterCollection();
```

ArgentSea provides a variety of extension methods to work with the parameters collection.

* Methods to easily add parameters to any parameters collection
* Methods to simplify obtaining values from parameters.
* Methods to Map Models properties to parameters.

### Creating Parameters with Extension Methods

ArgentSea offers a set of extension methods that simplify the code required to optimally create and populate parameters and also handle database nulls.

The methods to add parameters to a collection are provider-specific, since they are converting .NET types to database types. This means that the extension methods won’t appear unless you have a `using` statement referencing the provider.

## [SQL Server](#tab/tabid-sql)

````csharp
using ArgentSea.Sql;
````

> [!NOTE]
> The QueryParameterCollection and SqlParameterCollection inherit from the same base class. Not only can you use these methods on ArgentSea’s `QueryParameterCollection`, but you can also use them on the standard ADO.NET `SqlCommand.Parameters` property.

Here are some code examples:

````csharp
parameters.AddSqlIntInputParameter("@TransactionId", transactionId);
parameters.AddSqlDecimalInputParameter("@Amount", amount, 16, 2);
parameters.AddSqlNVarCharInputParameter("@Name", name, 255);
parameters.AddSqlFloatOutputParameter("@Temperature");
// These methods all support a fluent api, so these can be written instead as:
var parameters = new QueryParameterCollection()
    .AddSqlIntInputParameter("@TransactionId", transactionId)
    .AddSqlDecimalInputParameter("@Amount", amount, 16, 2)
    .AddSqlNVarCharInputParameter("@Name", name, 255)
    .AddSqlRealOutputParameter("@Temperature");
// And these methods also work on the data provider’s command parameters collection.
cmd.Parameters.AddSqlIntInputParameter("@TransactionId", transactionId)
    .AddSqlDecimalInputParameter("@Amount", amount, 16, 2)
    .AddSqlNVarCharInputParameter("@Name", name, 255)
    .AddSqlRealOutputParameter("@Temperature");
````

## [PostgreSQL](#tab/tabid-pg)

````csharp
using ArgentSea.Pg;
````

> [!NOTE]
> The QueryParameterCollection and NpgsqlParameterCollection inherit from the same base class. Not only can you use these methods on ArgentSea’s `QueryParameterCollection`, but you can also use them on the standard ADO.NET `NpgsqlCommand.Parameters` property.

```csharp
parameters.AddPgIntegerInputParameter("TransactionId", transactionId);
parameters.AddPgDecimalInputParameter("Amount", amount, 16, 4);
parameters.AddPgVarCharInputParameter("Name", name, 255);
parameters.AddPgDoubleOutputParameter("Temperature");
// These methods all support a fluent api, so these can be written instead as:
var parameters = new QueryParameterCollection()
    .AddPgIntegerIputnParameter("TransactionId", transactionId)
    .AddPgDecimalInputParameter("Amount", amount, 16, 2)
    .AddPgVarCharInputParameter("Name", name, 255)
    .AddPgDoubleOutputParameter("Temperature");
// And these methods also work on the data provider’s command parameters collection.
cmd.Parameters..AddPgIntegerInputParameter("TransactionId", transactionId)
    .AddPgDecimalInputParameter("Amount", amount, 16, 2)
    .AddPgVarCharInputParameter("Name", name, 255)
    .AddPgDoubleOutputParameter("Temperature");
```

***

Where appropriate, the methods have overloads that accept nullable value types. When the nullable type is null, the parameter will be set to a database Null value. If you are not using the Nullable overloads, then the values Guid.Empty, double.NaN, and float.NaN will also be saved as database Nulls. Likewise, null strings will be set to database Nulls, but empty strings will save as zero-length strings. The extension methods accepting string values have a max length argument, and those converting to Ansi database values have a code page parameter. The decimal methods have arguments for specifying precision and scale.

### Creating Parameters with the Mapper

The Mapper uses Model property attributes to automatically generate code that is much like what would be created in the previous section. Technically, the Mapper procedures are also extension methods, but we are discussing them separately in this section.

Assuming that the Model (in this example, a “Store” class) has Mapping attributes associated with each of its properties, you can render all the corresponding input parameters and set their respective values with:

```csharp
parameters.CreateInputParameters<Store>(customer, logger);
```

You can do something similar with output parameters — though it would be unlikely that you would want to want to create *only* output parameters. You will probably need at least one input parameter (likely a key). If you create the input parameter first, it will not be duplicated by the Mapper as it generates output parameters.

## [SQL Server](#tab/tabid-sql)

```csharp
parameters.AddSqlIntInputParameter("@TransactionId", transactionId);
parameters.CreateOutputParameters<Store>(logger);
// Again, these methods all support a fluent api, so this can be written instead as:
var parameters = new QueryParameterCollection()
    .AddSqlIntInputParameter("@TransactionId", transactionId)
    .CreateOutputParameters<Store>(logger);
```

## [PostgreSQL](#tab/tabid-pg)

```csharp
parameters.AddPgIntegerInputParameter("TransactionId", transactionId);
parameters.CreateOutputParameters<Store>(logger);
// Again, these methods all support a fluent api, so this can be written instead as:
var parameters = new QueryParameterCollection()
    .AddPgIntegerInputParameter("TransactionId", transactionId)
    .CreateOutputParameters<Store>(logger);
```

***

Finally, you can always add parameters using standard ADO.NET syntax:

```csharp
var parameter = new System.Data.SqlClient.SqlParameter();
parameter.SqlDbType = System.Data.SqlDbType.Int;
parameter.Value = transactionId;
command.Parameters.Add(parameter);
```

## Fetching Data

Retrieving database data consists of running a stored procedure or function on each connection. ArgentSea provides various methods to offer the best approach:

### Connection methods

Both database connections and shards have distinct Read and Write connections. The query methods are the same for both Read and Write connections, but the distinction allows the system to “scale out” reads and writes. The Read connection should be used for SELECT-only stored procedures or functions; the Write connection should be used for everything else.

Even if you do not currently have read-only endpoints, like mirrors or active standbys, consistent determination of Read and Write access will allow you to scale-out in the future. If you want to enforce this, you might consider placing read procedures and write procedures into different database schemas, then granting EXECUTE permission only the login corresponding to the Read or Write connection.

If only the Read or Write connection is set, the other connection will also have that same value.

These methods can be used on Read or Write connections on both a Database object or a single Shard instance within a ShardSet:

| Methods | Uses Mapper | Description |
| --- |:---:| --- |
| __LookupAsync__ |   | Returns a value from the database. This may be a return value (int) or single output parameter. |
| __RunAsync__ |   | Executes a database command. No results are returned. |
| __QueryAsync__ |   | Returns the typed object created by a handler delegate. |
| __MapListAsync__ |  • | Returns a List of typed objects from the data results. |
| __MapReaderAsync__ | • | Returns a typed object created by the [Mapper](/api/ArgentSea.Mapper.html) from __DataReader__ results. |
| __MapOutputAsync__ | • | Returns a typed object created by the [Mapper](/api/ArgentSea.Mapper.html) from output parameters *and* __DataReader__ results. |

### ShardSet methods

These methods execute the same command more-or-less concurrently on all the shards in a ShardSet. The ShardSet methods always use the Read connection.

| Methods | Uses Mapper | Description |
| --- |:---:| --- |
| __QueryAllAsync__ |   | Returns a List of any non-null objects created by a handler delegate in any shard. The list count will not be larger than the shard count. |
| __QueryFirstAsync__ |   | Returns the first non-null object created by a handler delegate in any shard. Use this when you are expecting a single result. |
| __MapListAsync__ |  • | Returns a *combined* List of typed objects from the data results, across all shards. Additional result sets will be discarded. |
| __MapReaderAllAsync__ | • | Returns a List including any non-null objects created from any shard’s data reader results (not using output parameters). The list count will not be larger than the shard count. |
| __MapReaderFirstAsync__ | • | Returns the first non-null object created on any shard where the procedure/function returns data reader results. Use this when you are expecting a single result and not using output parameters.  |
| __MapOutputAllAsync__ | • | Returns a List including any non-null objects that were created by invoking a procedure/function that returns results via output parameters *and* data reader results. The list count will not be larger than the shard count. |
| __MapOutputFirstAsync__ | • | Returns the first non-null object created by invoking a procedure/function that returns results via output parameters *and* data reader results. Use this when you are expecting a single result and using output parameters. |

### Method Arguments

The arguments are largely consistent across all of the methods.

#### Required Argument: (string) sprocName

This is simply the name of the stored procedure or function to be invoked. This string value is required for every data access method.

The general practice is to provide a stored procedure/function name with string constant, like this:

```csharp
await database.RunAsync("ws.MyProcedureName", parameters, cancellationToken);
```

As larger applications evolve, however, one can lose track of which database procedures are *actually being used* by the application. It is not unusual for a custom application to have hundreds of data procedures, only a fraction of which are used.  

Pro tip: you might consider consistently referencing procedure names via a static class, like this.

```csharp
internal static class DataProcedures
{
  //This should be a COMPREHENSIVE list of stored procedure names.
  //You can use the reference count to determine what is in use.
  public static string StoreAdd { get;  } = "ws.StoreAdd";
  public static string StoreList { get; } = "ws.StoreList";
  public static string StoreLocationGet { get; } = "ws.StoreLocationGet";
  public static string StoreLocationDetailsGet { get;  } = "ws.StoreLocationDetailsGet";
  public static string StoreLocationsAllByUser { get; } = "ws.StoreLocationsAllByUser";
  public static string StoreLocationsByGroupIDs { get; } = "ws.StoreLocationsByGroupIDs";
  // ...
}
// Now you can reference the procedure name like this:
await database.RunAsync(DataProcedures.StoreAdd, parameters, cancellationToken);

```

Centralizing the procedure name list allows reviewers to see which procedures are actually used by the application (and ensure that non are misspelled). Surfacing them using static properties, as in the example, allows Visual Studio to provide a “reference count” for each procedure (constants do not do this). When the method that called this stored procedure/function is removed from the code, the zero “reference count” will make it obvious that, even though the name defined, it is not actually used.

The reference count popup data even allows you to find all of the methods that invoke that stored procedure/function. This can be helpful as you prune different data procedures that seem to do the same thing.

#### Required Argument: (DbParameterCollection) parameters

The abstract `DbParameterCollection` is implemented by ArgentSea’s [QueryParameterCollection](/api/ArgentSea.QueryParameterCollection.html) object. Because it is also implemented by the provider-specific command.Parameters property, if you have a command with valid parameters defined (for some reason), you can use that too.

This value can be null if there are no parameters.

> [!WARNING]
> When working with output parameters in standard ADO.NET, you may habitually maintain a variable reference to any output parameters you created before adding it to the collection. This makes it easy to get the output parameter value after the query is executed.  
> This approach will not work with sharded data, because ArgentSea will *copy* the parameter set before executing the stored procedure/function. Any referenced output parameters will *not* contain a data result.

#### Optional ShardSet Argument: (IEnumerable&lt;ShardParameterValue&lt;TShard&gt;&gt;) shardParameterValues

Some shard query method overloads accept a `ShardParameterValue` object. This object allows you to specify which shards should be queried and even provide distinct parameter values to each shard.

For example, suppose your User record returns a list of “Friends”. The Friend detail data may be hosted on other shards, but not on *every* shard. Building a list of `ShardParameterValue` objects from the User results would limit the subsequent queries to just the relevant shards.

The `ShardParameterValue` type has a ShardId and an optional parameter name and value. Only shards with at least one listed ShardId will be queried. If a parameter name is also specified, the corresponding parameter will be set to that value on the indicated shard. You can include multiple parameters/values on the same shard by repeating the shardId.

#### Optional ShardSet Argument: (string) shardParameterName

Some shard query overloads also accept the name of the parameter that represents the name of the parameter that should be set to shardId value. If specified, ArgentSea will set this parameter value to the current shardId value as it executes each query.

For example, a query for a list of records that spans shards could be enhanced if the query new the value of its own ShardId. Alternatively, because a shard misconfiguration might result in catastrophic data corruption (due to the high likelihood of duplicate record identities between shards), you might require that stored procedures or functions that write to the database also have a ShardId parameter that they validate is correct.

#### Optional Argument: (QueryResultModelHandler&lt;TShard, TArg, TModel&gt;>) resultHandler

This is only used in the QueryAsync methods. As described earlier, the data query process is divided into two processes. The resultHandler is a delegate that may be invoked concurrently by distinct, shard-specific threads.

If you use a data access method prefixed with Map*, this argument is not required because the delegate provided by the Mapper is used. If the Mapper does not suit your purpose, then a custom delegate must be provided to a Query* method.

Your custom delegate can have an argument that provides additional data or context information. Information on how to build a custom delegate is provided below.

#### Optional Argument: (bool) isTopOne

Some overloads expose the isTopOne option, which allows a minor optimization when only a single result is expected. For example, if you are looking up a record by its key, you don’t need to allocate space for multiple results when only a single result can ever be returned.

#### Optional Argument: (TArg) optionalArgument

If you are creating a custom data handling method, you may need to provide additional data or context information. This argument may be generically typed. The provided object is passed to your result handling delegate.

#### Required Argument: (CancellationToken) cancellationToken

The cancellation token allow you to cancel asynchronous operations. ASP.NET MVC provides cancellation tokens and these can be passed along. In this way, when a user abandons their session, any uncompleted queries can be cancelled.

### The MapReader&ast; and MapOutput&ast; Methods

The __MapReader&ast;__ and __MapOutput&ast;__ methods are similar. Both use the Mapping attributes to resolve data to Model objects. The __MapOutput&ast;__ method uses *output parameters* to build the root result object; the __MapReader__ methods use a (single record) __DataReader__ result instead.

So, if you use output parameters (which is potentially more performant), use __MapOutput&ast;__. If you use standard SELECTs to return your data, use __MapReader&ast;__.

Both methods support multiple result sets that populate properties that contain Lists of related data. For example, you might have an Order record with a property containing an OrderItem List. The list items come from (additional) DataReader results. You can have up to eight of these List properties.

> [!TIP]
> The order in which your attribute-mapped class appears in the generic definitions should be the same order as the list data results in the procedure output.

An example of calling each would be:

```csharp
// In this example, ws.GetOrderDetails returns Order data in output parameters:
_database.MapOutputAsync<Order>("ws.GetOrderDetails", parameters, cancellation);
// Here, ws.GetOrderDetails returns simple Order data in a single-row SELECT:
_database.MapReaderAsync<Order>("ws.GetOrderDetails", parameters, cancellation);

// Now ws.GetOrderDetails returns Order data in output parameters and a list of OrderItem from a SELECT:
_database.MapOutputAsync<Order, OrderItems>("ws.GetOrderDetails", parameters, cancellation);
// Finally, ws.GetOrderDetails returns Order data in a single-row SELECT, then a list of OrderItems from a 2nd SELECT:
_database.MapReaderAsync<Order, Order, OrderItems>("ws.GetOrderDetails", parameters, cancellation);

// Expanding this, we now have output parameters and three SELECTs:
_database.MapOutputAsync<Store, OrderHistory, Locations, Contact>("ws.GetStoreDetails", parameters, cancellation);
// Likewise, the procedure now returns four SELECTs, and the third one is a single-row SELECT with the base customer data,
// the remaining select are used to build customer property lists (order history, locations, and contacts):
_database.MapReaderAsync<Store, OrderHistory, Store, Locations, Contact>("ws.GetStoreDetails", parameters, cancellation);
```

In both methods, the generic type in the *first* position is the return type. If additional results are included in the result stream, the subsequent types define the order in which they are expected in the DataReader results. You can have up to eight DataReader results streamed to distinct List properties.

* In the __MapOutput&ast;__ example, then, the result type is Order and the first DataReader result is a series of OrderItems.
* In the __MapOutput&ast;__ example, the result type is Order, and the first DataReader result is the Order data, and the second DataReader result is a series of OrderItems.

### The Query&ast; Methods

The __Query&ast;__ methods provide the most control, as you are given raw ADO.NET query results to construct whatever return value you like. You can return a list, a dictionary, or any type of Model object. When you call a __Query&ast;__ method, you must provide a handler method whose signature corresponds to the [QueryResultModelHandler](/api/ArgentSea.QueryResultModelHandler-3.html) delegate.

There are two obvious scenarios for the __Query&ast;__ methods:

* The Model class is defined in a library, so Mapping attributes cannot be added.
* The rendering a complex return value is beyond the capabilities of the Mapper.

The delegate even has a parameter that allows you to provide custom data (through the query method) with which to construct your result object.

The delegate must be thread-safe.  The [ShardSet](/api/ArgentSea.ShardSetsBase-2.ShardSet.html) manages the complexity of initializing multiple queries on multiple connections and multiple results, but it is the delegate that takes the database results (from each connection/thread) and creates an object result.

> [!NOTE]
> The Mapper provides several thread-safe, high-performance [QueryResultModelHandler](/api/ArgentSea.QueryResultModelHandler-3.html) delegates. In fact, providing a Mapper delegate to the __Query&ast;__ method is exactly how the __MapOutput&ast;__ an __MapOutput&ast;__ methods are implemented. You can use this yourself to extend the Mapper; just provide your own delegate that calls the Mapper in turn.

Details on implementing the [QueryResultModelHandler](/api/ArgentSea.QueryResultModelHandler-3.html) delegate is in the next section.

## Handling Data Results

If you are using data mapping attributes in your Model classes, the __MapReader&ast;__, __MapOutput&ast;__, and __MapList&ast;__ methods make handling data results unnecessary. This section is for queries that use the __Query&ast;__ methods, which allow you to return an arbitrary object from the data input.

If you are familiar with ADO.NET programming, this will be very familiar. The delegate simply receives the standard ADO.NET query results and processes them like it would in most other ADO.NET scenarios.

As an example, a method with the correct signature for returning a Store model looks like this:

```csharp
public static Store MyStoreHandler (
    short shardId,
    string sprocName,
    Department department, // this is an optional custom argument
    DbDataReader reader,
    DbParameterCollection parameters,
    string connectionDescription,
    ILogger logger)
{
    var result = new Store();
    // use the reader argument and/or parameters collection to set your result properties.
    return result;
}
```

### The Arguments

Both the return type (“Store”, in the example) and the optional data argument (“Department”, in the example) are generic, so they can be of any type.

#### (TShard) shardId

The shardId argument will be a default value, like null or zero, when not using a ShardSet; otherwise it will be set to the current ShardId. This value is essential when building ShardKey or ShardChild types, where the shard identity is a component of the record identity.

#### (string) sprocName

This is the name of the stored procedure or function that was executed. It is provided to the procedure for logging purposes.

#### (TArg) optionalArgument

The third argument type is a generic parameter; the type is defined when you declare the delegate. This object provides whatever external data or context that many be necessary or useful in order to create your result.

If it not needed (i.e. most cases), define the type as `object`. This allows you to use the __Query&ast;__ overloads that do not require this parameter; in those cases, this value will be null.

#### (DbDataReader) reader

The reader argument is a standard data reader. You can call `reader.MoveNext()` to get the next row and `reader.NextResult()` to get the next result set. You do not need to dispose of it when you are done.

#### (DbParameterCollection) parameters

The parameters collection contains the input and output parameters for the query. ArgentSea offers a set of extension methods to simplify converting parameter values to .NET types. These are extension methods on the parameter object (not the collection).

```csharp
var transactionId = parameters["@TransactionId"].GetInteger();
var amount = parameters["@Amount"].GetNullableDecimal();
var name = parameters["@Name"].GetString();
```

#### (string) connectionDescription

The connectionDescription argument allows the logger to include the connection that raised the error or event. You should include this (and also the stored/function procedure name) in any logging or errors in your procedures. Because your delegate could run on multiple connections, this can be essential debugging information.

#### (ILogger) logger

Finally, the logger argument allows you to write debugging, warning, and error information to the application logs. This is the same logger instance as is passed in through the various query methods.