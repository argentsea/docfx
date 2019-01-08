# Setting Parameters

With the ArgentSea framework, you need to set parameter values *before* a connection or command is created. The ADO.NET standard parameter collections cannot be created without a command object host. To fill this need, ArgentSea provides a [QueryParameterCollection](/api/ArgentSea.QueryParameterCollection.html) object, which is simply a collection of ADO.NET DbParameters. This object allow you to create an instance with a simple `new` statement.

```csharp
var parameters = new QueryParameterCollection();
```

ArgentSea provides a variety of extension methods to work with the parameters collection.

* Methods to easily add parameters to any parameters collection
* Methods to simplify obtaining values from parameters.
* Methods to Map Models properties to parameters.

## Creating Parameters with Extension Methods

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

## Creating Parameters with the Mapper

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

> [!TIP]
> As you define parameters in your stored procedures or functions, being as consistent as possible will make using the Mapper easy. For example, every time you save or retrieve a Model, you should always use the same parameters or fields results. Occiasionally missing parameters or columns will likely be maddening.

Next: [Fetching Data](fetching.md)
