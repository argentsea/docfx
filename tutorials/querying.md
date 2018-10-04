# Querying Data

This deep dive assumes that you have your project correctly [configured](configuration.md) and that you have some understanding of ArgentSea’s [Mapping](mapping.md) capabilities

ArgentSea was originally built to support application data sharding. Even if you do not use data sharding in your application, a brief discussion of the issues will help explain the architecture behind of ArgentSea’s data access approach. It will not be more difficult than any other ADO.NET query.

## Understanding Sharding

The best way to understand the query architecture of ArgentSea is to describe a typical ADO.NET query then describe how this must change to account for concurrent multi-threaded queries across a shard set. These changes are not complicated, and they should be helpful even if your project does use sharding.

A typical ADO.NET data access method follows these steps:

1. Start with a *connection* object, created from a connection string.
2. Create a *command* object that is associated with the *connection* object.
3. Next, the command's *Parameters* property is populated with the necessary input and output parameters.
4. The command is executed on the opened connection.
5. That data results are used to create and populate a Model object, which is used by the application.

In a sharded environment, however:

* The same parameters must be executed on multiple connections — reversing the steps 1 to 3.
* A distinct command object must be executed and the results processed on a separate thread for each connection. The parameters cannot be shared (different threads would overwrite each other’s values) and the result handler must be thread-safe because it could be simultaneously executing on different connections.

ArgentSea manages the challenges of multi-threaded access access with a four-step approach:

1. Declare the parameters and arguments that will be passed to the stored procedures.
2. Create a thread for each shard connection, then create the *connection* (and *command*) object for each.
3. Copy the parameter values to the parameter collection on each shard’s *command* object.
4. Call the stored procedure/function on each shard’s thread. When results are obtained, call (thread-safe) code to create and populate a Model object.
5. Combine the results and return them to the caller.

Ultimately, using ArgentSea on multiple shards is no more difficult than writing simple ADO.NET database access code (and usually much easier). Previously, your data access procedure would set the ADO.NET parameters, run the query, then convert the results to a Model object. Now you simply need to split that process into *two* procedures:

* One procedure that sets the parameters and calls an ArgentSea query method
* Another procedure that converts the results (data reader and/or output parameters) to a Model object result.

The second step can use standard ADO.NET code, or you can use ArgentSea’s optional [Mapping](mapping.md) functionality, which enables the Model creation process to be automatic.

This ArgentSea query paradigm applies even to non-sharded queries using the Databases collection. This provides some design consistency, but also enables the Mapper for both sharded and non-sharded data.

## Setting Parameters

With the ArgentSea framework, you need to set parameter values *before* a connection or command is created. The ADO.NET standard parameter collections cannot be created without a command object host. Consequently, ArgentSea provides a QueryParameter object, which is simply a collection of ADO.NET DbParameters. You can create an instance with a simple `new` statement.

```C#
var prms = new QueryParameterCollection();
```

ArgentSea provides a variety of extension methods to work with the parameters collection.

* Methods to easily add parameters to a collection
* Methods to simplify obtaining values from parameters.
* Methods to Map Models properties to parameters.

### Creating Parameters with Extension Methods

ArgentSea offer a set of extension methods that simplify the code required to optimally create and populate parameters and also handle database nulls.

The methods to add parameters to a collection are provider-specific, since they are converting .NET types to database types. This means that the extension methods won’t appear unless you have a `using` statement referencing ArgentSea.Sql or ArgentSea.Pg.

> [!NOTE]
> Because the QueryParameterCollection inherits from the same base class as SqlParameterCollection and NpgsqlParameterCollection, all of ArgentSea’s extension methods will work on any of them. In other words, you can also use these methods on either the `QueryParameterCollection` or on the `Parameters` property of the *command* object.

Here are some examples:

## [SQL Server](#tab/tabid-sql)

````C#
prms.AddSqlIntInParameter("@TransactionId", transactionId);
prms.AddSqlDecimalInParameter("@Amount", amount, 16, 2);
prms.AddSqlNVarCharInParameter("@Name", name, 255);
prms.AddSqlFloatOutParameter("@Temperature");
// These methods all support a fluent api, so these can be written instead as:
var prms = new QueryParameterCollection()
    .AddSqlIntInParameter("@TransactionId", transactionId)
    .AddSqlDecimalInParameter("@Amount", amount, 16, 2)
    .AddSqlNVarCharInParameter("@Name", name, 255)
    .AddSqlRealOutParameter("@Temperature");
// And these methods also work on the data provider’s command parameters collection.
cmd.Parameters.AddSqlIntInParameter("@TransactionId", transactionId)
    .AddSqlDecimalInParameter("@Amount", amount, 16, 2)
    .AddSqlNVarCharInParameter("@Name", name, 255)
    .AddSqlRealOutParameter("@Temperature");
````

## [PostgreSQL](#tab/tabid-pg)

```C#
prms.AddPgIntegerInParameter("TransactionId", transactionId);
prms.AddPgDecimalInParameter("Amount", amount, 16, 4);
prms.AddPgVarCharInParameter("Name", name, 255);
prms.AddPgDoubleOutParameter("Temperature");
// These methods all support a fluent api, so these can be written instead as:
var prms = new QueryParameterCollection()
    .AddPgIntegerInParameter("TransactionId", transactionId)
    .AddPgDecimalInParameter("Amount", amount, 16, 2)
    .AddPgVarCharInParameter("Name", name, 255)
    .AddPgDoubleOutParameter("Temperature");
// And these methods also work on the data provider’s command parameters collection.
cmd.Parameters..AddPgIntegerInParameter("TransactionId", transactionId)
    .AddPgDecimalInParameter("Amount", amount, 16, 2)
    .AddPgVarCharInParameter("Name", name, 255)
    .AddPgDoubleOutParameter("Temperature");
```

***

Where appropriate, the methods have overloads that accept nullable value types. When the nullable type is null, the parameter will be set to a database Null value. If you are not using the Nullable overloads, then the values Guid.Empty, double.NaN, and float.NaN will also be saved as database Nulls. Likewise, null strings will be set to database Nulls, but empty strings will save as zero-length strings. The extension methods accepting string values have a max length argument, and those converting to Ansi database values have a code page parameter. The decimal methods have arguments for specifying precision and scale.

### Creating Parameters with the Mapper

The Mapper uses Model property attributes to automatically generate code that is much like what would be created in the previous section. Technically, the Mapper procedures are also extension methods, but we are discussing them separately in this section.

Assuming that the Model (in this example, a “Customer” class) has Mapping attributes associated with each of its properties, you can render all the corresponding input parameters and set their respective values with:

```C#
prms.MapToInParameters<Customer>(customer, logger);
```

You can do somthing similar with output parameters, but it would be unlikely that you would want to want to create *only* output parameters. You will probably need at least one input parameter (a key, probably). If you create the input parameter first, it will not be duplicated by the Mapper as it generates output parameters.

## [SQL Server](#tab/tabid-sql)

```C#
prms.AddSqlIntInParameter("@TransactionId", transactionId);
prms.MapToOutParameters<Customer>(logger);
// Again, these methods all support a fluent api, so this can be written instead as:
var prms = new QueryParameterCollection()
    .AddSqlIntInParameter("@TransactionId", transactionId)
    .MapToOutParameters<Customer>(logger);
```

## [PostgreSQL](#tab/tabid-pg)

```C#
prms.AddPgIntegerInParameter("TransactionId", transactionId);
prms.MapToOutParameters<Customer>(logger);
// Again, these methods all support a fluent api, so this can be written instead as:
var prms = new QueryParameterCollection()
    .AddPgIntegerInParameter("TransactionId", transactionId)
    .MapToOutParameters<Customer>(logger);
```

***

Finally, you can always add parameters using standard ADO.NET syntax:

```C#
var prm = new System.Data.SqlClient.SqlParameter();
prm.SqlDbType = System.Data.SqlDbType.Int;
prm.Value = transactionId;
cmd.Parameters.Add(prm);
```

## Fetching Data

Retrieving database data consists of running a stored procedure or function on each connection. ArgentSea provides various methods to offer the best approach:

### Database methods

These methods can be used on Database connections, and on a shard instance (read connection or write connection) within a ShardSet:

| Methods | Uses Mapper | Description |
| --- |:---:| --- |
| __ListAsync__ |  • | Returns a List of typed objects from the data results. Additional result sets will be discarded. |
| __LookupAsync__ | • | Returns a value from the database. This may be a return value (int) or single output parameter. |
| __RunAsync__ | • | Executes a database command. No results are returned. |
| __QueryAsync__ |   | Returns the typed object created by a handler delegate. |
| __ReadAsync__ | • | Returns a typed object created from data reader results. |
| __GetOutAsync__ | • | Returns a typed object created from output parameters *and* data reader results. |

### ShardSet methods

These methods execute the same command more-or-less concurrently on all the shards in a ShardSet. The ShardSet methods always use the Read connection.

| Methods | Uses Mapper | Description |
| --- |:---:| --- |
| __ListAsync__ |  • | Returns a *combined* List of typed objects from the data results, across all shards. Additional result sets will be discarded. |
| __QueryAllAsync__ |   | Returns a List of any non-null objects created by a handler delegate in any shard. The list count will not be larger than the shard count. |
| __QueryFirstAsync__ |   | Returns the first non-null object created by a handler delegate in any shard. Use this when you are expecting a single result. |
| __ReadAllAsync__ | • | Returns a List including any non-null objects created from any shard’s data reader results (not using output parameters). The list count will not be larger than the shard count. |
| __ReadFirstAsync__ | • | Returns the first non-null object created on any shard where the procedure/function returns data reader results. Use this when you are expecting a single result and not using output parameters.  |
| __GetOutAllAsync__ | • | Returns a List including any non-null objects that were created by invoking a procedure/function that returns results via output parameters *and* data reader results. The list count will not be larger than the shard count. |
| __GetOutFirstAsync__ | • | Returns the first non-null object created by invoking a procedure/function that returns results via output parameters *and* data reader results. Use this when you are expecting a single result and using output parameters. |

#### The Read* and GetOut* Methods

The __Read__ and __GetOut__ methods are very similar. Both user the Mapping attributes. The __GetOut__ method uses *output parameters* to build the root result object; the __Read__ methods use a (single record) DataReader result instead. If you use output parameters (which is potentially more performant), use __GetOut__.

Both methods support properties that contain Lists of further data. For example, you might have an Order record with a property containing an OrderItem List. The list items come from (additional) DataReader results. You can have up to eight of these List properties.

An example of calling each would be:

```C#
// In this example, ws.GetOrderDetails returns Order data in output parameters:
_database.GetOutAsync<Order>("ws.GetOrderDetails", prms, cancellation);
// Here, ws.GetOrderDetails returns simple Order data in a single-row SELECT:
_database.ReadAsync<Order>("ws.GetOrderDetails", prms, cancellation);

// Now ws.GetOrderDetails returns Order data in output parameters and a list of OrderItem from a SELECT:
_database.GetOutAsync<Order, OrderItems>("ws.GetOrderDetails", prms, cancellation);
// Finally, ws.GetOrderDetails returns Order data in a single-row SELECT, then a list of OrderItems from a 2nd SELECT:
_database.ReadAsync<Order, Order, OrderItems>("ws.GetOrderDetails", prms, cancellation);

// Expanding this, we now have output parameters and three SELECTs:
_database.GetOutAsync<Customer, OrderHistory, Locations, Contact>("ws.GetCustomerDetails", prms, cancellation);
// Likewise, the procedure now returns four SELECTs, and the third one is a single-row SELECT with the base customer data,
// the remaining select are used to build customer property lists (order history, locations, and contacts):
_database.ReadAsync<Customer, OrderHistory, Customer, Locations, Contact>("ws.GetCustomerDetails", prms, cancellation);
```

In both methods, the generic type in the *first* position is the return type. If additional results are included in the result stream, the subsequent types define the order in which they are expected in the DataReader results. You can have up to eight DataReader results streamed to distinct List properties.

* In the __GetOut__ example, then, the result type is Order and the first DataReader result is a series of OrderItems.
* In the __Read__ example, the result type is Order, and the first DataReader result is the Order data, and the second DataReader result is a series of OrderItems.

#### The Query* Methods

The __Query__ methods provide the most control, as you are given raw ADO.NET query results to construct whatever return value you like. The handler procedure must have a method signature that corresponds to the `QueryResultModelHandler` delegate.

There are two obvious scenarios for the __Query__ methods:

* The Model class is defined in a (sealed) library, so Mapping attributes cannot be added.
* The return value is complex and beyond the capabilities of the Mapper.

The delegate even has a parameter that allows you to provide custom data (through the query method) with which to construct your result object.




In other words, the [ShardSet](/api/ArgentSea.ShardSetsBase-2.ShardSet.html) manages the complexity of initializing multiple queries on multiple connections and multiple results, but it is the delegate that takes the database results (from each connection/thread) and creates an object result.

> [!NOTE]
> The Mapper provides several thread-safe, high-performance `QueryResultModelHandler` delegates. In fact, providing a Mapper delegate to the __Query__ method is exactly how the __Read__ an __GetOut__ methods are implemented.

### The Optional Arguments

#### int shardParameterOrdinal 

(ShardSet only)

#### bool isTopOne

#### TArg optionalArgument

#### IEnumerable<TShard> exclude

## Handling Data Results



The methods to convert data types to .NET types are extension methods on the parameter object (not the collection).

```C#
var transactionId = prms["@TransactionId"].GetInteger();
var amount = prms["@Amount"].GetNullableDecimal();
var name = prms["@Name"].GetString();
```


When you execute the query, you pass in the parameters you set, and ArgentSea will run your second

The second section needs to conform to a delegate method signature that can be invoked by the framework. 



