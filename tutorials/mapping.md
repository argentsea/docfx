# Mapping Deep-Dive

The Mapper make data-access coding simpler and more productive by using  property attributes to map a model class’s properties to data values — parameters, reader columns, and (in the case of SQL Server) table-value parameters. This reduces and simplifies the amount of code required to render data.

## Overview

Using the Mapper consists of two parts, which should be familiar from other scenarios:

* Use property attributes to define how each property in your model class should be mapped to a data store (if at all)
* Call a method to which uses data results and the attribute metadata to populate the properties

By defining metadata about the names of parameters or result columns, the Mapper can automatically map properties to columns and/or parameters. Several query methods on both Database connections and ShardSets implicitly use the Mapper.

## Property Attributes

You use properties attributes to define the metadata that the Mapper requires. For example, given this very simple model class:

```cs
using System;

public class Subscriber
{
    public int SubscriberId { get; set; }

    public string Name { get; set; }

    public DateTime Expiration { get; set; }
}
```

Adding mapping attributes to this class provides the metadata to automatically map these properties to stored procedures:

## [SQL Server](#tab/tabid-sql)

```cs
using System;
using ArgentSea.Sql;

public class Subscriber
{
    [MapToSqlInt("@SubID", true)]
    public int SubscriberId { get; set; }

    [MapToSqlNVarChar("@SubscriberName", 255)]
    public string Name { get; set; }

    [MapToSqlDateTime2("@EndDate")]
    public DateTime Expiration { get; set; }
}
```

The “@” parameter prefix is optional — ArgentSea will add the “@” automatically for parameters and remove it automatically when reading data reader rows.

## [PostgreSQL](#tab/tabid-pg)

```cs
using System;
using ArgentSea.Pg;

public class Subscriber
{
    [MapToPgInteger("SubId", true)]
    public int SubscriberId { get; set; }

    [MapToPgVarChar("SubscriberName", 255)]
    public string Name { get; set; }

    [MapToPgTimestamp("EndDate")]
    public DateTime Expiration { get; set; }
}
```

***

Often, due to different naming conventions or development drift, database column names and the corresponding .NET properties names do not match. That is why every attribute requires a “name” argument — which should correspond to the database name. The Mapper will create query parameters and reference DataReader columns based on this name.

> [!IMPORTANT]
> Database parameters and columns should be named as consistently as possible. In most cases, this means the parameters have the same name as the columns they reference. If you like to use varying parameter names or alias columns in your result, you will find the Mapper unhelpful.

Properties without a mapping attribute are simply ignored.

### Attribute Types

A mapping attribute is defined for most common database types. Attributes for spatial data types, CLR types, XML, and JSON types (for example) are missing because there is no straightforward mapping between the core .NET base types and these database types. ArgentSea supports writing a custom handler to render any of these complex types; such work is no more difficult than writing the same processing in ADO.NET.

The attribute itself defines the underlying database type. Naturally, the attribute type and the property type must match. For example, a `long` (Int64) property *must* map to a `bigint` database type. The Mapper will throw an error if these types do not match. There is no attempt to cast data to a different type, even if the cast would be successful.

Many data attribute types have an additional parameters. The *length* argument, for example, on string and array types, helps optimize data access performance by ensuring that buffers are sized appropriately.

Here is catalog of the current attributes, along with their arguments and corresponding .NET types:

# [SQL Server](#tab/tabid-sql)

| Attribute | Arguments | .NET types | SqlType |
| --- |--- | --- | --- |
| MapToSqlNVarCharAttribute | length¹ | String, Enum², Nullable&lt;Enum&gt; | NVarChar |
| MapToSqlNCharAttribute | length | String, Enum², Nullable&lt;Enum&gt; |  NChar |
| MapToSqlVarCharAttribute | length¹, localeid³ | String, Enum², Nullable&lt;Enum&gt; | VarChar |
| MapToSqlCharAttribute | length, localeid³ | String, Enum², Nullable&lt;Enum&gt; | Char |
| MapToSqlBigIntAttribute | | Int64, Enum⁴, Nullable&lt;Int64&gt;, Nullable&lt;Enum&gt; | BigInt |
| MapToSqlIntAttribute | | Int32, Enum⁴, Nullable&lt;Int32&gt;, Nullable&lt;Enum&gt;| Int |
| MapToSqlSmallIntAttribute | | Int16, Enum⁴, Nullable&lt;Int16&gt;, Nullable&lt;Enum&gt; | SmallInt |
| MapToSqlTinyIntAttribute | | Byte, Enum⁴, Nullable&lt;Byte&gt;, Nullable&lt;Enum&gt; | TinyInt |
| MapToSqlBitAttribute | | Boolean, Nullable&lt;Boolean&gt; | Bit |
| MapToSqlDecimalAttribute | precision, scale | Decimal, Nullable&lt;Decimal&gt; | Decimal |
| MapToSqlMoneyAttribute | | Decimal, Nullable&lt;Decimal&gt; | Money |
| MapToSqlSmallMoneyAttribute | | Decimal, Nullable&lt;Decimal&gt; | SmallMoney |
| MapToSqlFloatAttribute | | Double, Nullable&lt;Double&gt; |Float |
| MapToSqlRealAttribute | | Float, Nullable&lt;Float&gt; | Real |
| MapToSqlDateTimeAttribute | | DateTime, Nullable&lt;DateTime&gt; | DateTime |
| MapToSqlDateTime2Attribute | precision | DateTime, Nullable&lt;DateTime&gt; | DateTime2 |
| MapToSqlDateAttribute | | DateTime, Nullable&lt;DateTime&gt; | Date |
| MapToSqlTimeAttribute | | TimeSpan, Nullable&lt;TimeSpan&gt; | Time |
| MapToSqlDateTimeOffsetAttribute | | DateTimeOffset, Nullable&lt;DateTimeOffset&gt; | DateTimeOffset |
| MapToSqlVarBinaryAttribute | length¹ | byte[] | VarBinary |
| MapToSqlBinaryAttribute |  length | byte[] | Binary |
| MapToSqlUniqueIdentifierAttribute | | Guid, Nullable&lt;Guid&gt; | UniqueIdentifier |

¹ For “max” values (nvarchar(max), varbinary(max), etc.) use length of -1.

² The Enum name is saved as string.

³ Locale Id is the Ansi code page to use for Unicode conversion. For en-US locale, for example, use 1033.

⁴ The Enum value is saved based on its underlying numeric value. The Enum integer *base type* (int, short, byte, etc.) must match the database type.

# [PostgreSQL](#tab/tabid-pg)

| Attribute | Arguments | .NET types | SQL Type |
| --- |--- | --- | --- |
| MapToPgVarCharAttribute | length | String, Enum¹, Nullable&lt;Enum&gt; | VarChar |
| MapToPgCharAttribute | length | String, Enum¹, Nullable&lt;Enum&gt; |  Char |
| MapToPgTextAttribute | | String, Enum¹, Nullable&lt;Enum&gt; | VarChar |
| MapToPgBigintAttribute | | Int64, Enum², Nullable&lt;Int64&gt;, Nullable&lt;Enum&gt; | Bigint |
| MapToPgIntegerAttribute | | Int32, Enum², Nullable&lt;Int32&gt;, Nullable&lt;Enum&gt; | Integer |
| MapToPgSmallintAttribute | | Int16, Enum², Nullable&lt;Int16&gt;, Nullable&lt;Enum&gt; | Smallint |
| MapToPgInternalCharAttribute | | Byte, Enum², Nullable&lt;Byte&gt;, Nullable&lt;Enum&gt; | InternalChar |
| MapToPgBooleanAttribute | | Boolean, Nullable&lt;Boolean&gt; | Boolean |
| MapToPgNumericAttribute | precision, scale | Decimal, Nullable&lt;Decimal&gt; | Numeric |
| MapToPgMoneyAttribute | | Decimal, Nullable&lt;Decimal&gt; | Money |
| MapToPgDoubleAttribute | | Double, Nullable&lt;Double&gt; |Double |
| MapToPgRealAttribute | | Float, Nullable&lt;Float&gt; | Real |
| MapToPgTimestampAttribute | | DateTime, DateTimeOffset, Nullable&lt;DateTime&gt;, Nullable&lt;DateTimeOffset&gt; | Timestamp |
| MapToPgTimestampTzAttribute | | DateTimeOffset, Nullable&lt;DateTimeOffset&gt; | TimestampTz |
| MapToPgDateAttribute | | DateTime, Nullable&lt;DateTime&gt; | Date |
| MapToPgTimeAttribute  | | TimeSpan, Nullable&lt;TimeSpan&gt; | Time |
| MapToPgIntervalAttribute | | TimeSpan, Nullable&lt;TimeSpan&gt; | Interval |
| MapToPgTimeTzAttribute | | TimeSpan, DateTimeOffset, Nullable&lt;TimeSpan&gt;, Nullable&lt;DateTimeOffset&gt; | TimeTz |
| MapToPgArrayAttribute | | Array | Array |
| MapToPgByteaAttribute | length | byte[] | Bytea |
| MapToPgHstoreAttribute | length | IDictionary&lt;string, string&gt; | Hstore |
| MapToPgUuidAttribute | | Guid, Nullable&lt;Guid&gt; | UniqueIdentifier |

¹ The Enum name is saved as string.

² The Enum value is saved based on its underlying numeric value. The Enum integer *base type* (int, short, byte, etc.) must match the database type.

***

#### Required

Finally, the the data attributes have an optional `required` (sic) parameter. If a database field is DbNull, the Mapper will normally set the corresponding property to null. However, the missing value may represent an entirely absent record. In this case, the correct result is a *null* object, not a valid instance with null/default properties. Setting a property attribute’s `required` argument to True causes the Mapper to return a null object if the property would be null. By default (if not specified), `required` is false.

### Handling Nulls and Empty Types

Because the Mapper is handling database values, there is generally a possibility that the database value is DbNull. How this is converted to a .NET type depends upon the type.

#### Strings and Arrays

A .NET string with a value of *null* or a null array will be saved as a DbNull. Empty strings will save as a zero-length string.

#### Integers

Integers cannot be null, so the advent of nullable types is a godsend for mapping to database storage. To save or retrieve an integer (byte, Int16, Int32, or Int64) database value from a column that allows null, you should declare a nullable value type.

#### Floating Point Numbers

Like integer types, floating point types (Double and Float) can be wrapped in a nullable value. However, ArgentSea also handles *NaN* as a DbNull. If the floating point value is presented as a nullable type, then ArgentSea will save or retrieve NaN; if floating point type is presented, then NaN will be converted to/from a DbNull.

#### Guids

Rather like floating point types, Guid.Empty (00000000-0000-0000-0000-000000000000) will be converted to a data DbNull when read from or written to the database. Also like floats, if you need to write an empty Guid value, wrap it in a nullable type.

#### Enum Types

.NET enum values can be stored as either numbers or strings. Writing to a text column will automatically save the *name* of the enum; writing to a numeric column saves the *number* value.

> [!WARNING]
> Enums can inherit from several base types (byte, short, int, etc.). If you are saving to a numeric database column, the base type must correctly correspond to the database data type. Enums are Int32 by default.

Nullable Enum types will read or write as a DbNull when the value is null.

#### ShardKey and ShardChild

These are special types and are discussed in detail in the [sharding](sharding.md) section.

#### The MapToModel attribute

Complex object models may include properties that are objects with their *own* properties, which also need to be mapped to the underlying data.

For example, you might have an Address object that you use for Customers, Vendors, Contacts, Stores, and more. The Vendor class, then, has a property of type Address, and the Customer class has an Address property too. Since the address information is included with the results from the database, the Mapper should map the matching values to the Address object. The `MapToModel` attribute tells the Mapper to do this.

Of course, the property’s type must also have data mapping attributes on the appropriate class properties. The type referenced by a MapToModel attribute can *itself* have a object property with a MapToModel attribute.

In other words, a *Store* object can have a property of type *Address*, which might in turn have a property of type *Coordinates*. If the *Coordinates* type has two properties, each with a `MapToDouble` attribute, the Mapper will be able to map the Latitude and Longitude values to the *Store.Address.Latitude* and *Store.Address.Longitude* fields respectively.

Properties with the MapToModel attribute cannot be null, so the root object must be instantiate all of its properties when it is created.

## Mapping Targets

The ArgentSea Mapper maps to:

* Query input parameters
* Query output parameters
* Data reader columns
* Table-valued parameters (SQL Server only)

The mapper does *not* generate dynamic SQL statements. The Mapper may be useful in situations where dynamic SQL is used, but, philosophically, this is not encouraged. Stored procedures are generally more secure, more performant, and offer a less tightly-coupled architecture.

### The Parameter Collection

The Mapper’s parameter methods are implemented as an extension method to the (abstract) DbParametersCollection, which is inherited by each provider implementation of the DbCommand.Parameters property. This means that you can call the Mapper through the command object of any provider.

```cs
cmd.Parameters.MapInputParameters<MyDataClass>(myDataClass, logger);
// or
cmd.Parameters.MapCreateOutputParameters<MyDataClass>(logger);
```

These extension methods can be combined with the other extension methods for a *fluent API*, which allows you to build a logical sequence of code that may be more readable.

# [SQL Server](#tab/tabid-sql)

For example, this code uses the fluent API to to set an output parameter and then the mapper to create and set all the other object properties from a transaction object.

```cs
cmd.Parameters.AddSqlIntOutParameter("@TransactionId")
    .MapInputParameters<Transaction>(transaction, logger);
```

# [PostgreSQL](#tab/tabid-pg)

For example, this code uses the fluent API to to set an output parameter and then the mapper to create and set all the other object properties from a transaction object.

```cs
cmd.Parameters.AddPgIntegerOutParameter("TransactionId")
    .MapInputParameters<Transaction>(transaction, logger);
```

***

In ADO.NET, you normally access the data parameters collection (`SqlParametersCollection`, `NpgsqlParametersCollection`, etc.) through the `Parameters` property of the command object. In ArgentSea, you can still do this; the Mapper and other extension methods work on the parameters collection property. 

When working with sharded data, however, this presents a problem that is described in detail in the tutorial on [querying](querying.md). The gist is that there is a need to create a parameters list independently of a command object.

Enter the `QueryParameterCollection` class. It’s functionally not much more than a parameter list, but it can be created without a command object. Because it also inherits from the abstract *DbParameterCollection*, the same extension methods — like the Mapper — work on this object too.

## [SQL Server](#tab/tabid-sql)

Again, the optional fluent API makes setting an input parameter and a mapped set of output parameters quite simple:

```cs
var prms = new QueryParameterCollection()
    .AddSqlBigIntInParameter("@ID", _id)
    .MapCreateOutputParameters<MyClass>(logger);
```

## [PostgreSQL](#tab/tabid-pg)

Again, the optional fluent API makes setting an input parameter and a mapped set of output parameters quite simple:

```cs
var prms = new QueryParameterCollection()
    .AddPgBigintInParameter("ID", _id)
    .MapCreateOutputParameters<MyClass>(logger);
```

***

### Mapping to Input Parameters

You can create input parameters with the `MapToInputParameters` method. The mapping attributes in your class will be used to:

* Create the set of input parameters
* Set the value of those parameters to the value of the corresponding property.

That is all the Mapper does. The Mapper simply saved you the time and effort of hand-coding a whole bunch of parameters. You can view the parameters in the debugger and you can add, remove or update any of them. If a particular query needs a parameter that is not presented in a property attribute, just add it to parameter the collection yourself!

Any parameters already added to the parameter collection will not be recreated (the names must match exactly). This is helpful if you need to treat one or more parameters differently (say, an output parameter in a collection of input parameters). 

If you don’t want the Mapper to create a particular parameter set, you can provide a list of parameter names to suppress.

### Mapping to Output Parameters

Working with output parameters is done in two steps:

* Create the output parameters before executing the query
* Read the output parameter values after executing the query

## [SQL Server](#tab/tabid-sql)

This example creates and sets the CustomerId parameter and creates all of the output parameters. It then executes the query and reads the output values into a new object.

```cs
cmd.Parameters.AddSqlIntInParameter("@CustomerId", _Id)
    .MapCreateOutputParameters<Customer>(logger);
await cmd.ExecuteNonQueryAsync();
var customer = cmd.Parameters.ReadOutParameters<Customer>(logger);
```

## [PostgreSQL](#tab/tabid-pg)

This example creates and sets the CustomerId parameter and creates all of the output parameters. It then executes the query and reads the output values into a new object.

```cs
cmd.Parameters.AddPgIntegerInParameter("CustomerId", _Id)
    .MapCreateOutputParameters<Customer>(logger);
await cmd.ExecuteNonQueryAsync();
var customer = cmd.Parameters.ReadOutParameters<Customer>(logger);
```

***

Of course, it would be quite unusual to have a query that *only* uses output parameters. Because the input parameter is added to the collection first, the output parameter will be automatically skipped. As with input parameters, you can also provide a list of parameter names that you want to explicitly skip. And also like input parameters, the `MapToOutputParameters` method simply creates output parameters; you can modify the collection as needed.

Once the parameters are set and the procedure is executed, the Mapper can read the values of the output parameters into the corresponding properties of a new object instance. The `ReadOutputParameters` method returns a new object with the properties set.

> [!NOTE]
> The `MapOutput&ast;` methods of the Database or ShardSet objects fetch results from the database and implicitly use the Mapper to return a Model based upon output parameters. In most cases, you would use one of those methods rather than `ReadOutParameters` on the Mapper directly.

### The DataReader

The Mapper also converts the rows presented by a DataReader object into a list of corresponding objects, or a single row into a Model instance. 

For example, to map to a list of objects:

```cs
var customers = rdr.MapToList<Customer>(logger);
```

The IList result will contain an object instance for each valid row. If an attribute is marked “required” but the corresponding data field is DbNull, then the object will not be included listed results.

To map to a single Model instance:

````cs
var customer = rdr.MapToModel<Customer>(logger);
````

> [!NOTE]
> As with output parameters, the `MapReader&ast;` or `MapListAsync` methods of the Database or ShardSet objects fetch results from the database and implicitly use the Mapper to return a Model based upon DataReader results. In most cases, you would use one of those methods rather than `MapToList` or `MapToModel` on the Mapper.

## Performance

The ArgentSea Mapper is written to be as high-performance as optimized hand-coded data access code. However, there is a hitch.

Property attributes can only be retrieved using *reflection*, which is relatively slow .NET code. To avoid this type of performance penalty on every data access, ArgentSea uses reflection only the first time the mapping is performed; using that metadata it then creates and compiles an “Expression Tree”to build an optimized, compiled mapping. The compiled code is cached in memory and reused for all subsequent calls.

> [!WARNING]
> The Mapper will be relatively slow (and CPU intensive) the *first time* each model class is mapped to parameters or data. The initial compilation usually takes less than a second. Subsequent calls will execute the data to property mapping at native machine-code speeds. When the application is restarted, the memory is cleared and the compilation overhead occurs again.

## Logging

You have surely noticed that every Mapper command requires a logger instance — an object that implements the `ILogger` interface. A supportable application requires logging, so the parameter is not optional. The .NET Core environment provides objects that log to the console, debug window, Windows event logs, file system, [Application Insights](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-asp-net-core), [CloudWatch](https://github.com/aws/aws-logging-dotnet#aspnet-core-logging), and much more. ArgentSea can consume any of these logging providers and provide diagnostic and runtime data to their respective targets.

In production, you will generally want to use log level *Information*. In development you may find *Debug* or even *Trace* very helpful.

> [!CAUTION]
> Be sure to manage the logging level in your configuration. This determines the amount of logging and this can have a substantial impact upon performance.

### Logging Levels

The logging levels determine the types of events that are logged. These are described below:

### Critical

Logs when the circuit breaker is triggered on a connection or command. This may generate many downstream errors until the functionality is restored.

#### Error

In most cases, an error condition will throw to the caller so they become the caller’s responsibility to handle or log. Because data access may happen on multiple threads, however, a simple throw may lose context. If the data reader passed to the Mapper is closed or null, this is logged as an exception along with the connection description.

#### Warning

ArgentSea creates a warning log event when starting an automatic retry on a connection or command.

#### Information

When the circuit breaker is triggered, ArgentSea creates a log record each time a test transaction is attempted and again when functionality is restored.

#### Debug

The logged events in the Debug level are intended to help diagnose internal processes that may not be returning the expected results.

The first type of event is when a DbNull value is presented to an object that then becomes null or empty, which happens with ShardKey, ShardChild, or any object with a Required argument set to true. When this happens unexpectedly, it can be difficult to determine which database value caused the problem (as now *no* properties exist to determine the culprit). This logging event identifies which DbNull caused the result to be null or Empty.

The second type of event provides full visibility into the generated code used to build the Mapper’s activity. The Expression Tree is walked and the pseudo-code saved to the log before it is compiled. This can be extremely useful in understanding the complexities of the Mapping behavior. The log record will be rather long and the extraction may not be efficient, but it also runs only during the first data access.

This log level also reports when a parameter attribute was defined but the parameter was not found among the output parameters. This might be by design or it might be a programming oversight.

Finally, the Mapper logs when it did not find an cached delegate so an Expression Tree is being built and compiled. This is normal at startup because the cache will be empty; if these event occur afterward, there is likely a problem.

#### Trace

The Mapper creates a trace log record as it iterates over properties. This can provide insight into the current context when other error conditions occur. Also, the logger will report the execution time for commands sent to a database connection or shard sets.