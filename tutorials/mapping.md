# Mapping Deep-Dive

The Mapper make data-access coding simpler and more productive by using  property attributes to map a model class’s properties to data values — parameters, reader columns, and (in the case of SQL Server) table-value parameters. This reduces and simplifies the amount of code required.

## Overview

Using the Mapper consists of two steps:

* Define how each property in your model class should be mapped to a data store (if at all) using property attributes
* Call a Mapper method to automatically create data parameters, DataReader handler, etc.

### Property Attributes

You use properties attributes to define the metadata that the Mapper requires. For example, given this very simple model class:

```C#
using System;

public class Subscriber
{
    public int SubscriberId { get; set; }

    public string Name { get; set; }

    public DateTime Expiration { get; set; }
}
```

Adding mapping attributes to this class provides the metadata to automatically map these properties to stored procedures:

# [SQL Server](#tab/tabid-sql)

```C#
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

# [PostgreSQL](#tab/tabid-pg)

```C#
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

Often, due to different naming conventions or development drift, database column names and the corresponding .NET properties names do not match. That is why every attribute requires a “name” argument — which should largly correspond to the database name. The Mapper will create query parameters based on this name and reference DataReader columns with this name.

> [!WARNING]
> Database parameters and columns should be named as consistently as possible. In most cases, this means the parameters have the same name as the columns they reference. If you like to alias columns or abbreviate parameters, you will find the Mapper largely unhelpful.


## Attribute Types

There is a mapping attribute defined for most common database types. Spatial data types, CLR types, XML, and JSON types are examples of missing attributes. These are absent because there is not a straightforward mapping between the core .NET base types and these database types. It is very possible to write custom handlers to render this from your database; indeed it is not much harder than writing the necessary custom code without this framework.

The attribute itself defines the underlying database type. Naturally, the attribute type and the property type must match. For example, a `long` (Int64) property *must* map to a `bigint` database type. The Mapper will throw an error if these types do not match. There is no attempt to cast data to a different type, even if the cast would be successful.

Many data attribute types have an additional parameters. The *length* argument, for example, helps optimize data access performance by ensuring that buffers are sized appropriately.

Here is catalog of the current attributes, along with their arguments and corresponding .NET types:

### [SQL Server](#tab/tabid-sql)

| Attribute | Arguments | .NET types | SqlType |
| --- |--- | --- | --- |
| MapToSqlNVarCharAttribute | length¹ | String, Enum², Nullable\<Enum\> | NVarChar |
| MapToSqlNCharAttribute | length | String, Enum², Nullable\<Enum\> |  NChar |
| MapToSqlVarCharAttribute | length¹, localeid³ | String, Enum², Nullable\<Enum\> | VarChar |
| MapToSqlCharAttribute | length, localeid³ | String, Enum², Nullable\<Enum\> | Char |
| MapToSqlBigIntAttribute | | Int64, Enum⁴, Nullable\<Int64\>, Nullable\<Enum\> | BigInt |
| MapToSqlIntAttribute | | Int32, Enum⁴, Nullable\<Int32\>, Nullable\<Enum\>| Int |
| MapToSqlSmallIntAttribute | | Int16, Enum⁴, Nullable\<Int16\>, Nullable\<Enum\> | SmallInt |
| MapToSqlTinyIntAttribute | | Byte, Enum⁴, Nullable\<Byte\>, Nullable\<Enum\> | TinyInt |
| MapToSqlBitAttribute | | Boolean, Nullable\<Boolean\> | Bit |
| MapToSqlDecimalAttribute | precision, scale | Decimal, Nullable\<Decimal\> | Decimal |
| MapToSqlMoneyAttribute | | Decimal, Nullable\<Decimal\> | Money |
| MapToSqlSmallMoneyAttribute | | Decimal, Nullable\<Decimal\> | SmallMoney |
| MapToSqlFloatAttribute | | Double, Nullable\<Double\> |Float |
| MapToSqlRealAttribute | | Float, Nullable\<Float\> | Real |
| MapToSqlDateTimeAttribute | | DateTime, Nullable\<DateTime\> | DateTime |
| MapToSqlDateTime2Attribute | precision | DateTime, Nullable\<DateTime\> | DateTime2 |
| MapToSqlDateAttribute | | DateTime, Nullable\<DateTime\> | Date |
| MapToSqlTimeAttribute | | TimeSpan, Nullable\<TimeSpan\> | Time |
| MapToSqlDateTimeOffsetAttribute | | DateTimeOffset, Nullable\<DateTimeOffset\> | DateTimeOffset |
| MapToSqlVarBinaryAttribute | length¹ | byte[] | VarBinary |
| MapToSqlBinaryAttribute |  length | byte[] | Binary |
| MapToSqlUniqueIdentifierAttribute | | Guid, Nullable\<Guid\> | UniqueIdentifier |

¹ For “max” values (nvarchar(max), varbinary(max), etc.) use length of -1.

² The Enum name is saved as string.

³ Locale Id is the Ansi code page to use for Unicode conversion. For en-US locale, for example, use 1033.

⁴ The Enum value is saved based on its underlying numeric value. The Enum integer *base type* (int, short, byte, etc.) must match the attribute type.

### [PostgreSQL](#tab/tabid-pg)

| Attribute | Arguments | .NET types | SQL Type |
| --- |--- | --- | --- |
| MapToPgVarCharAttribute | length | String, Enum¹, Nullable\<Enum\> | VarChar |
| MapToPgCharAttribute | length | String, Enum¹, Nullable\<Enum\> |  Char |
| MapToPgTextAttribute | | String, Enum¹, Nullable\<Enum\> | VarChar |
| MapToPgBigintAttribute | | Int64, Enum², Nullable\<Int64\>, Nullable\<Enum\> | Bigint |
| MapToPgIntegerAttribute | | Int32, Enum², Nullable\<Int32\>, Nullable\<Enum\> | Integer |
| MapToPgSmallintAttribute | | Int16, Enum², Nullable\<Int16\>, Nullable\<Enum\> | Smallint |
| MapToPgInternalCharAttribute | | Byte, Enum², Nullable\<Byte\>, Nullable\<Enum\> | InternalChar |
| MapToPgBooleanAttribute | | Boolean, Nullable\<Boolean\> | Boolean |
| MapToPgNumericAttribute | precision, scale | Decimal, Nullable\<Decimal\> | Numeric |
| MapToPgMoneyAttribute | | Decimal, Nullable\<Decimal\> | Money |
| MapToPgDoubleAttribute | | Double, Nullable\<Double\> |Double |
| MapToPgRealAttribute | | Float, Nullable\<Float\> | Real |
| MapToPgTimestampAttribute | | DateTime, DateTimeOffset, Nullable\<DateTime\>, Nullable\<DateTimeOffset\> | Timestamp |
| MapToPgTimestampTzAttribute | | DateTimeOffset, Nullable\<DateTimeOffset\> | TimestampTz |
| MapToPgDateAttribute | | DateTime, Nullable\<DateTime\> | Date |
| MapToPgTimeAttribute  | | TimeSpan, Nullable\<TimeSpan\> | Time |
| MapToPgIntervalAttribute | | TimeSpan, Nullable\<TimeSpan\> | Interval |
| MapToPgTimeTzAttribute | | TimeSpan, DateTimeOffset, Nullable\<TimeSpan\>, Nullable\<DateTimeOffset\> | TimeTz |
| MapToPgArrayAttribute | | Array | Array |
| MapToPgByteaAttribute | length | byte[] | Bytea |
| MapToPgHstoreAttribute | length | IDictionary\<string, string\> | Hstore |
| MapToPgUuidAttribute | | Guid, Nullable\<Guid\> | UniqueIdentifier |

¹ The Enum name is saved as string.

² The Enum value is saved based on its underlying numeric value. The Enum integer *base type* (int, short, byte, etc.) must match the attribute type.

***

### Required

Finally, the the data attributes have an *optional* `required` parameter. It defaults to false. Generally, you might set a key value to “required”; this tell the Mapper that if the value is null, then the entire record is missing and the object result should be null.

For example, suppose you are using a query’s output parameters to fetch a record. A null value in the key field means that the record doesn’t exist. Returning a valid new object with a null properties isn’t the correct behavior; the Mapper should return a null object.

When using replicated or mirrored data, there can be latency between the primary and cloned databases. In high-performance scenarios, you may split your reads and writes and your application can attempt reads that *should* be successful but fail because of this latency. In this case, the `required` parameter enables the system to detect an unexpectedly missing record on a clone or mirror and retry on the master or primary.

When `required` is set to true, then:

* The Mapper will return a null object if this parameter or column is null.
* The shard set will retry a data fetch on the a Write connection.



### Enum Types

.NET enum values can be stored with their base integer data type (int, short, etc.), or as their string name, or (PostgreSQL only) as an Enum database type. 

* If the mapping attribute database type is a numeric database type, then the base number type is stored
* If the mapping attribute database type is a text-type field, the Mapper stores the name of the enum value
* If the mapping attribute database type is Enum (PostgreSQL only), then the Enum value is set

### Nullable

### ShardKey and ShardChild

## Mapping Attributes

## [SQL Server](#tab/tabid-sql)

### SQL Server Mapping Attributes


To map a property

## [PostgreSQL](#tab/tabid-pg)

### PostgreSQL Mapping Attributes

There is a mapping attribute defined for most SQL Server data types. The principals exceptions are the CLR types and complex types, like hierarchyid, geometry, geography, variant, and xml. The property type must correspond

***


## Mapping Targets

The ArgentSea Mapper maps to:

* Query input parameters
* Query output parameters
* Data reader columns
* Table-value parameters (SQL Server only)

The mapper does not generate dynamic SQL statements. The Mapper may be useful in situations where dynamic SQL is used, but philosophically this is not encouraged. Stored procedures are generally more secure, more performant, and offer a less tightly-coupled architecture.



### Mapping to Input Parameters


### Mapping to Output Parameters


### The DataReader


## The Implicit use of the Mapper

## Required Attributes


## Performance

The ArgentSea Mapper is written to be as high-performance as optimized hand-coded data access code. However, there is a hitch.

Property attributes can only be retrieved using *reflection*, which is relatively slow .NET code. To avoid this type of performance penalty on every data access, ArgentSea uses reflection only the first time the mapping is performed; using that metadata it then creates and compiles an “Expression Tree”to build an optimized, compiled mapping. The compiled code is cached in memory and reused for all subsequent calls.

> [!INFORMATION]
> The Mapper will be relatively slow (and CPU intensive) the *first time* each model class is mapped to parameters or data. The initial compilation usually takes about a second or less. Subsequent calls will execute the data to property mapping at native machine-code speeds. When the application is restarted, the memory is cleared and the compilation overhead occurs again.

## Instrumentation



The Mapper also use