# Setting Parameters

With the ArgentSea framework, you need to set parameter values *before* a connection or command is created. The ADO.NET standard parameter collections cannot be created without a command object host. To fill this need, ArgentSea provides a [QueryParameterCollection](/api/ArgentSea.QueryParameterCollection.html) object, which is simply a collection of ADO.NET DbParameters. This object allow you to create an instance with a simple `new` statement.

```csharp
var parameters = new QueryParameterCollection();
```

ArgentSea provides a variety of extension methods to work with the parameters collection.

* Methods to easily add parameters to any parameters collection
* Methods to simplify obtaining values from parameters.
* Methods to Map Models properties to parameters.

> [!NOTE]
> Parameter names are implicitly normalized. On SQL Server this means always ensuring a “@” prefix. On PostgreSQL, future versions might automatically convert to lowercase.

## Creating Parameters with Extension Methods

ArgentSea offers a set of extension methods that simplify the code required to optimally create and populate parameters and also handle database nulls.

The methods to add parameters to a collection are provider-specific, since they are converting .NET types to database types. This means that the extension methods won’t appear unless you have a `using` statement referencing the provider.

## [SQL Server](#tab/tabid-sql)

> [!NOTE]
> The QueryParameterCollection and SqlParameterCollection inherit from the same base class. Not only can you use these methods on ArgentSea’s `QueryParameterCollection`, but you can also use them on the standard ADO.NET `SqlCommand.Parameters` property.

To access the extension methods, you need the following `using` statement:

```csharp
using ArgentSea.Sql;
```

Adding a correctly typed parameter now requires only one line:

```csharp
parameters.AddSqlIntInputParameter("@TransactionId", transactionId);
parameters.AddSqlDecimalInputParameter("@Amount", amount, 16, 2);
parameters.AddSqlNVarCharInputParameter("@Name", name, 255);
parameters.AddSqlFloatOutputParameter("@Temperature");
```

These methods all support a fluent api, so these can be written instead as:

```csharp
var parameters = new QueryParameterCollection()
    .AddSqlIntInputParameter("@TransactionId", transactionId)
    .AddSqlDecimalInputParameter("@Amount", amount, 16, 2)
    .AddSqlNVarCharInputParameter("@Name", name, 255)
    .AddSqlRealOutputParameter("@Temperature");
```

And these methods also work directly on the data provider’s command parameters collection.

```csharp
cmd.Parameters.AddSqlIntInputParameter("@TransactionId", transactionId)
    .AddSqlDecimalInputParameter("@Amount", amount, 16, 2)
    .AddSqlNVarCharInputParameter("@Name", name, 255)
    .AddSqlRealOutputParameter("@Temperature");
```

## [PostgreSQL](#tab/tabid-pg)

> [!NOTE]
> The QueryParameterCollection and NpgsqlParameterCollection inherit from the same base class. Not only can you use these methods on ArgentSea’s `QueryParameterCollection`, but you can also use them on the standard ADO.NET `NpgsqlCommand.Parameters` property.

To access the extension methods, you need the following `using` statement:

```csharp
using ArgentSea.Pg;
```

Adding a correctly typed parameter now requires only one line:

```csharp
parameters.AddPgIntegerInputParameter("transactionid", transactionId);
parameters.AddPgDecimalInputParameter("amount", amount, 16, 4);
parameters.AddPgVarcharInputParameter("name", name, 255);
parameters.AddPgDoubleOutputParameter("temperature");
```

These methods all support a fluent api, so these can be written instead as:

```csharp
var parameters = new QueryParameterCollection()
    .AddPgIntegerInputParameter("transactionid", transactionId)
    .AddPgDecimalInputParameter("amount", amount, 16, 2)
    .AddPgVarcharInputParameter("name", name, 255)
    .AddPgDoubleOutputParameter("temperature");
```

And these methods also work directly on the data provider’s command parameters collection.

```csharp
cmd.Parameters..AddPgIntegerInputParameter("transactionid", transactionId)
    .AddPgDecimalInputParameter("amount", amount, 16, 2)
    .AddPgVarcharInputParameter("name", name, 255)
    .AddPgDoubleOutputParameter("temperature");
```

***

Where appropriate, the methods have overloads that accept nullable value types. When the nullable type is null, the parameter will be set to a database Null value. If you are not using the Nullable overloads, then the values Guid.Empty, double.NaN, and float.NaN will also be saved as database Nulls. Likewise, null strings will be set to database Nulls, but empty strings will save as zero-length strings. The extension methods accepting string values have a max length argument, and those converting to Ansi database values have a code page parameter. The decimal methods have arguments for specifying precision and scale.

## Creating Parameters with the Mapper

The Mapper uses Model property attributes to automatically generate code that is much like what would be created in the previous section.

Assuming that the Model (in this example, a “Store” class) has Mapping attributes associated with each of its properties, you can render all the corresponding input parameters and set their respective values with:

```csharp
parameters.CreateInputParameters<Store>(store, logger);
```

This one line of code is much easier than individually coding the creation of each parameter and setting each value.


## [SQL Server](#tab/tabid-sql)

You can do something similar with output parameters. Because you will probably need at least one input parameter (likely a key), set that first. Then, when you generate the output parameters the Mapper will not duplicate this existing parameter with a redundant output parameter.

```csharp
parameters.AddSqlIntInputParameter("@TransactionId", transactionId);
parameters.CreateOutputParameters<Store>(logger);
// Again, these methods all support a fluent api, so this can be written instead as:
var parameters = new QueryParameterCollection()
    .AddSqlIntInputParameter("@TransactionId", transactionId)
    .CreateOutputParameters<Store>(logger);
```

## [PostgreSQL](#tab/tabid-pg)

Although ArgentSea can set and retrieve PostgreSQL output parameters, it is [not a recommended practice](http://www.npgsql.org/doc/basic-usage.html#stored-functions-and-procedures).

If you are determined to use output parameters, here is an example:

```csharp
parameters.AddPgIntegerInputParameter("transactionid", transactionId);
parameters.CreateOutputParameters<Store>(logger);
// Again, these methods all support a fluent api, so this can be written instead as:
var parameters = new QueryParameterCollection()
    .AddPgIntegerInputParameter("transactionid", transactionId)
    .CreateOutputParameters<Store>(logger);
```

***

Of course, you can always add parameters using standard ADO.NET syntax.

## [SQL Server](#tab/tabid-sql)

```csharp
var parameter = new System.Data.SqlClient.SqlParameter();
parameter.SqlDbType = System.Data.SqlDbType.Int;
parameter.Value = transactionId;
command.Parameters.Add(parameter);
```

## [PostgreSQL](#tab/tabid-pg)

```csharp
var parameter = new Npgsql.NpgsqlParameter();
parameter.NpgsqlDbType = NpgsqlType.Integer;
parameter.Value = transactionId;
command.Parameters.Add(parameter);
```

***

This may be useful for types not yet supported by ArgentSea.

> [!TIP]
> As you define parameters in your queries, naming them as consistently as possible will make using the Mapper easy.

Next: [Fetching Data](fetching.md)
