# Mapping Targets

The ArgentSea Mapper maps to:

* Query input parameters
* Query output parameters
* Data reader columns
* Table-valued parameters (SQL Server) or Copy statements (PostgreSQL)

The mapper does *not* generate dynamic SQL statements. The Mapper may be useful in situations where dynamic SQL is used, but, philosophically, this is not encouraged. Stored procedures and parameterized SQL statements are generally more secure, more performant, and offer a less tightly-coupled architecture.

## The Parameter Collection

The Mapper’s parameter methods are implemented as an extension method to the (abstract) DbParametersCollection, which is inherited by each provider implementation of the DbCommand.Parameters property. This means that you can call the Mapper through the command object of any provider.

```csharp
cmd.Parameters.CreateInputParameters<MyDataClass>(myDataClass, logger);
// or
cmd.Parameters.CreateOutputParameters<MyDataClass>(logger);
```

These extension methods can be combined with the other extension methods for a *fluent API*, which allows you to build a logical sequence of code that may be more readable.

# [SQL Server](#tab/tabid-sql)

For example, this code uses the fluent API to to set an output parameter and then the mapper to create and set all the other object properties from a transaction object.

```csharp
cmd.Parameters.AddSqlIntOutputParameter("@TransactionId")
    .CreateInputParameters<Transaction>(transaction, logger);
```

# [PostgreSQL](#tab/tabid-pg)

For example, this code uses the fluent API to to set an output parameter and then the mapper to create and set all the other object properties from a transaction object.

```csharp
cmd.Parameters.AddPgIntegerOutputParameter("TransactionId")
    .CreateInputParameters<Transaction>(transaction, logger);
```

***

In ADO.NET, you normally access the data parameters collection (`SqlParametersCollection`, `NpgsqlParametersCollection`, etc.) through the *Parameters* property of the command object. In ArgentSea, you can still do this; the Mapper and other extension methods work on the parameters collection property. 

When working with sharded data, however, this presents a problem that is described in detail in the tutorial on [querying](../querying/querying.md). The gist is that there is a need to create a parameters list independently of a command object.

Enter the [QueryParameterCollection](/api/ArgentSea.QueryParameterCollection.html) class. It’s functionally not much more than a parameter list, but it can be created without a command object. Because it also inherits from the abstract *DbParameterCollection*, the same extension methods — like the Mapper — work on this object too.

## [SQL Server](#tab/tabid-sql)

Again, the optional fluent API makes setting an input parameter and a mapped set of output parameters quite simple:

```csharp
var prms = new QueryParameterCollection()
    .AddSqlBigIntInputParameter("@ID", _id)
    .CreateOutputParameters<MyClass>(logger);
```

## [PostgreSQL](#tab/tabid-pg)

Again, the optional fluent API makes setting an input parameter and a mapped set of output parameters quite simple:

```csharp
var prms = new QueryParameterCollection()
    .AddPgBigintInputParameter("ID", _id)
    .CreateOutputParameters<MyClass>(logger);
```

***

## Mapping to Input Parameters

You can create input parameters with the `CreateInputParameters` method. This is an extension method on the parameters collection. The mapping attributes in your class will be used to:

* Create the set of input parameters
* Set the value of those parameters to the value of the corresponding property.

That is all the Mapper does. The Mapper simply saved you the time and effort of hand-coding a whole bunch of parameters. You can view the parameters in the debugger and you can add, remove or update any of them. If a particular query needs a parameter that is not presented in a property attribute, just add it to parameter the collection yourself!

Any parameters already added to the parameter collection will not be recreated (the names must match exactly). This is helpful if you need to treat one or more parameters differently (say, an output parameter in a collection of input parameters).

If you don’t want the Mapper to create a particular parameter set, you can provide a list of parameter names to suppress.

## Mapping to Output Parameters

Working with output parameters is done in two steps:

* Create the output parameters before executing the query
* Read the output parameter values after executing the query

## [SQL Server](#tab/tabid-sql)

This example creates and sets the CustomerId parameter and creates all of the output parameters. It then executes the query and reads the output values into a new object.

```csharp
cmd.Parameters.AddSqlIntInputParameter("@CustomerId", _Id)
    .CreateOutputParameters<Customer>(logger);
await cmd.ExecuteNonQueryAsync();
var customer = cmd.Parameters.ToModel<Customer>(logger);
```

## [PostgreSQL](#tab/tabid-pg)

This example creates and sets the CustomerId parameter and creates all of the output parameters. It then executes the query and reads the output values into a new object.

```csharp
cmd.Parameters.AddPgIntegerInputParameter("CustomerId", _Id)
    .CreateOutputParameters<Customer>(logger);
await cmd.ExecuteNonQueryAsync();
var customer = cmd.Parameters.ToModel<Customer>(logger);
```

***

Of course, it would be quite unusual to have a query that *only* uses output parameters. Because the input parameter is added to the collection first, the output parameter will be automatically skipped. As with input parameters, you can also provide a list of parameter names that you want to explicitly skip. And also like input parameters, the `CreateOutputParameters` method simply creates output parameters; you can modify the collection as needed.

Once the parameters are set and the query is executed, the Mapper can read the values of the output parameters into the corresponding properties of a new object instance. The `ToModel` method returns a new object with the properties set.

> [!NOTE]
> The `MapOutput*;` methods of the Database or ShardSet objects fetch results from the database and implicitly use the Mapper to return a Model based upon output parameters. In most cases, you would use one of those methods rather than `ToModel` on the Mapper directly.

## Mapping from DataReader Results

The Mapper also converts the rows presented by a DataReader object into a list of corresponding objects, or a single row into a Model instance. 

For example, to map to a list of objects:

```csharp
var customers = rdr.ToList<Customer>(logger);
```

The IList result will contain an object instance for each valid row. If an attribute is marked “required” but the corresponding data field is DbNull, then the object will not be included listed results.

To map to a single Model instance:

````csharp
var customer = rdr.ToModel<Customer>(logger);
````

> [!NOTE]
> As with output parameters, the `MapReader*;` or `MapListAsync` methods of the Database or ShardSet objects fetch results from the database and implicitly use the Mapper to return a Model based upon DataReader results. In most cases, you would use one of those methods rather than `ToList` or `ToModel` on the Mapper.

The DataReader mapping methods allow you to use multiple SELECT result to map both the base object and one or more list properties. The order of the generic objects provided to the Mapper determines the expected order of the result streams in the DataReader.

## Mapping to Data Loaders

In addition to the capabilities just discussed, ArgentSea has provider-specific Mapping functionality.

## [SQL Server](#tab/tabid-sql)

The SQL Server library offers a Mapper to set Table Valued Parameters. A Table Valued Parameter allows a series of records to be sent to a stored procedure, which can dramatically improve performance compared to multiple requests.

## [PostgreSQL](#tab/tabid-pg)

The PostgreSQL library offers a Mapper to create tables and load data using PostgreSQL COPY command and the NpgsqlBinaryImporter.

***

Next: [Logging](logging.md)