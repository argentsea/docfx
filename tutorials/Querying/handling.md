# Handling Data Results

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

## The Arguments

Both the return type (“Store”, in the example) and the optional data argument (“Department”, in the example) are generic, so they can be of any type.

### (TShard) shardId

The shardId argument will be a default value, like null or zero, when not using a ShardSet; otherwise it will be set to the current ShardId. This value is essential when building ShardKey or ShardChild types, where the shard identity is a component of the record identity.

### (string) sprocName

This is the name of the stored procedure or SQL statement that was executed. It is provided to the procedure for logging purposes.

### (TArg) optionalArgument

The third argument type is a generic parameter; the type is defined when you declare the delegate. This object provides whatever external data or context that many be necessary or useful in order to create your result.

If it not needed (i.e. most cases), define the type as `object`. This allows you to use the __Query&ast;__ overloads that do not require this parameter; in those cases, this value will be null.

### (DbDataReader) reader

The reader argument is a standard data reader. You can call `reader.MoveNext()` to get the next row and `reader.NextResult()` to get the next result set. You do not need to dispose of it when you are done.

### (DbParameterCollection) parameters

The parameters collection contains the input and output parameters for the query. ArgentSea offers a set of extension methods to simplify converting parameter values to .NET types. These are extension methods on the parameter object (not the collection).

```csharp
var transactionId = parameters["@TransactionId"].GetInteger();
var amount = parameters["@Amount"].GetNullableDecimal();
var name = parameters["@Name"].GetString();
```

### (string) connectionDescription

The connectionDescription argument allows the logger to include the connection that raised the error or event. You should include this (and also the stored procedure or statement name) in any logging or errors in your procedures. Because your delegate could run on multiple connections, this can be essential debugging information.

### (ILogger) logger

Finally, the logger argument allows you to write debugging, warning, and error information to the application logs. This is the same logger instance as is passed in through the various query methods.

Next: [Sharding](../sharding/sharding.md)
