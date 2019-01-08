# Property Attributes

You use properties attributes to define the metadata that the Mapper requires. For example, given this very simple model class:

```csharp
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

```csharp
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

```csharp
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

## Attribute Types

A mapping attribute is defined for most common database types. Attributes for spatial data types, CLR types, XML, and JSON types (for example) are missing because there is no straightforward mapping between the core .NET base types and these database types. ArgentSea supports writing a custom handler to render any of these complex types; such work is no more difficult than writing the same processing in ADO.NET.

The attribute itself defines the underlying database type. Naturally, the attribute type and the property type must match. For example, a `long` (Int64) property *must* map to a `bigint` database type. The Mapper will throw an error if these types do not match. There is no attempt to cast data to a different type, even if the cast would be successful.

Many data attribute types have an additional parameters. The *length* argument, for example, on string and array types, helps optimize data access performance by ensuring that buffers are sized appropriately.

Here is catalog of the current attributes, along with their arguments and corresponding .NET types:

# [SQL Server](#tab/tabid-sql)

| Attribute | Arguments | .NET types | SqlType |
| --- |--- | --- | --- |
| [MapToSqlNVarCharAttribute](/api-sql/ArgentSea.Sql.MapToSqlNVarCharAttribute.html) | length¹ | String, Enum², Nullable&lt;Enum&gt; | NVarChar |
| [MapToSqlNCharAttribute](/api-sql/ArgentSea.Sql.MapToSqlNCharAttribute.html) | length | String, Enum², Nullable&lt;Enum&gt; |  NChar |
| [MapToSqlVarCharAttribute](/api-sql/ArgentSea.Sql.MapToSqlVarCharAttribute.html) | length¹, localeid³ | String, Enum², Nullable&lt;Enum&gt; | VarChar |
| [MapToSqlCharAttribute](/api-sql/ArgentSea.Sql.MapToSqlCharAttribute.html) | length, localeid³ | String, Enum², Nullable&lt;Enum&gt; | Char |
| [MapToSqlBigIntAttribute](/api-sql/ArgentSea.Sql.MapToSqlBigIntAttribute.html) | | Int64, Enum⁴, Nullable&lt;Int64&gt;, Nullable&lt;Enum&gt; | BigInt |
| [MapToSqlIntAttribute](/api-sql/ArgentSea.Sql.MapToSqlIntAttribute.html) | | Int32, Enum⁴, Nullable&lt;Int32&gt;, Nullable&lt;Enum&gt;| Int |
| [MapToSqlSmallIntAttribute](/api-sql/ArgentSea.Sql.MapToSqlSmallIntAttribute.html) | | Int16, Enum⁴, Nullable&lt;Int16&gt;, Nullable&lt;Enum&gt; | SmallInt |
| [MapToSqlTinyIntAttribute](/api-sql/ArgentSea.Sql.MapToSqlTinyIntAttribute.html) | | Byte, Enum⁴, Nullable&lt;Byte&gt;, Nullable&lt;Enum&gt; | TinyInt |
| [MapToSqlBitAttribute](/api-sql/ArgentSea.Sql.MapToSqlBitAttribute.html) | | Boolean, Nullable&lt;Boolean&gt; | Bit |
| [MapToSqlDecimalAttribute](/api-sql/ArgentSea.Sql.MapToSqlDecimalAttribute.html) | precision, scale | Decimal, Nullable&lt;Decimal&gt; | Decimal |
| [MapToSqlMoneyAttribute](/api-sql/ArgentSea.Sql.MapToSqlMoneyAttribute.html) | | Decimal, Nullable&lt;Decimal&gt; | Money |
| [MapToSqlSmallMoneyAttribute](/api-sql/ArgentSea.Sql.MapToSqlSmallMoneyAttribute.html) | | Decimal, Nullable&lt;Decimal&gt; | SmallMoney |
| [MapToSqlFloatAttribute](/api-sql/ArgentSea.Sql..html) | | Double, Nullable&lt;Double&gt; |Float |
| [MapToSqlRealAttribute](/api-sql/ArgentSea.Sql.MapToSqlRealAttribute.html) | | Float, Nullable&lt;Float&gt; | Real |
| [MapToSqlDateTimeAttribute](/api-sql/ArgentSea.Sql.MapToSqlDateTimeAttribute.html) | | DateTime, Nullable&lt;DateTime&gt; | DateTime |
| [MapToSqlDateTime2Attribute](/api-sql/ArgentSea.Sql.MapToSqlDateTime2Attribute.html) | precision | DateTime, Nullable&lt;DateTime&gt; | DateTime2 |
| [MapToSqlDateAttribute](/api-sql/ArgentSea.Sql.MapToSqlDateAttribute.html) | | DateTime, Nullable&lt;DateTime&gt; | Date |
| [MapToSqlTimeAttribute](/api-sql/ArgentSea.Sql.MapToSqlTimeAttribute.html) | | TimeSpan, Nullable&lt;TimeSpan&gt; | Time |
| [MapToSqlDateTimeOffsetAttribute](/api-sql/ArgentSea.Sql.MapToSqlDateTimeOffsetAttribute.html) | | DateTimeOffset, Nullable&lt;DateTimeOffset&gt; | DateTimeOffset |
| [MapToSqlVarBinaryAttribute](/api-sql/ArgentSea.Sql.MapToSqlVarBinaryAttribute.html) | length¹ | byte[] | VarBinary |
| [MapToSqlBinaryAttribute](/api-sql/ArgentSea.Sql.MapToSqlBinaryAttribute.html) |  length | byte[] | Binary |
| [MapToSqlUniqueIdentifierAttribute](/api-sql/ArgentSea.Sql.MapToSqlUniqueIdentifierAttribute.html) | | Guid, Nullable&lt;Guid&gt; | UniqueIdentifier |

¹ For “max” values (nvarchar(max), varbinary(max), etc.) use length of -1.

² The Enum name is saved as string.

³ Locale Id is the Ansi code page to use for Unicode conversion. For en-US locale, for example, use 1033.

⁴ The Enum value is saved based on its underlying numeric value. The Enum integer *base type* (int, short, byte, etc.) must match the database type.

# [PostgreSQL](#tab/tabid-pg)

| Attribute | Arguments | .NET types | SQL Type |
| --- |--- | --- | --- |
| [MapToPgVarcharAttribute](/api-pg/ArgentSea.Pg.MapToPgVarCharAttribute.html) | length | String, Enum¹, Nullable&lt;Enum&gt; | VarChar |
| [MapToPgCharAttribute](/api-pg/ArgentSea.Pg.MapToPgCharAttribute.html) | length | String, Enum¹, Nullable&lt;Enum&gt; |  Char |
| [MapToPgTextAttribute](/api-pg/ArgentSea.Pg.MapToPgTextAttribute.html) | | String, Enum¹, Nullable&lt;Enum&gt; | VarChar |
| [MapToPgBigintAttribute](/api-pg/ArgentSea.Pg.MapToPgBigintAttribute.html) | | Int64, Enum², Nullable&lt;Int64&gt;, Nullable&lt;Enum&gt; | Bigint |
| [MapToPgIntegerAttribute](/api-pg/ArgentSea.Pg.MapToPgIntegerAttribute.html) | | Int32, Enum², Nullable&lt;Int32&gt;, Nullable&lt;Enum&gt; | Integer |
| [MapToPgSmallintAttribute](/api-pg/ArgentSea.Pg.MapToPgSmallintAttribute.html) | | Int16, Enum², Nullable&lt;Int16&gt;, Nullable&lt;Enum&gt; | Smallint |
| [MapToPgInternalCharAttribute](/api-pg/ArgentSea.Pg.MapToPgInternalCharAttribute.html) | | Byte, Enum², Nullable&lt;Byte&gt;, Nullable&lt;Enum&gt; | (Internal) "char"³ |
| [MapToPgBooleanAttribute](/api-pg/ArgentSea.Pg.MapToPgBooleanAttribute.html) | | Boolean, Nullable&lt;Boolean&gt; | Boolean |
| [MapToPgNumericAttribute](/api-pg/ArgentSea.Pg.MapToPgNumericAttribute.html) | precision, scale | Decimal, Nullable&lt;Decimal&gt; | Numeric |
| [MapToPgMoneyAttribute](/api-pg/ArgentSea.Pg.MapToPgMoneyAttribute.html) | | Decimal, Nullable&lt;Decimal&gt; | Money |
| [MapToPgDoubleAttribute](/api-pg/ArgentSea.Pg.MapToPgDoubleAttribute.html) | | Double, Nullable&lt;Double&gt; |Double |
| [MapToPgRealAttribute](/api-pg/ArgentSea.Pg.MapToPgRealAttribute.html) | | Float, Nullable&lt;Float&gt; | Real |
| [MapToPgTimestampAttribute](/api-pg/ArgentSea.Pg.MapToPgTimestampAttribute.html) | | DateTime, DateTimeOffset, Nullable&lt;DateTime&gt;, Nullable&lt;DateTimeOffset&gt; | Timestamp |
| [MapToPgTimestampTzAttribute](/api-pg/ArgentSea.Pg.MapToPgTimestampTzAttribute.html) | | DateTimeOffset, Nullable&lt;DateTimeOffset&gt; | TimestampTz |
| [MapToPgDateAttribute](/api-pg/ArgentSea.Pg.MapToPgDateAttribute.html) | | DateTime, Nullable&lt;DateTime&gt; | Date |
| [MapToPgTimeAttribute](/api-pg/ArgentSea.Pg.MapToPgTimeAttribute.html)  | | TimeSpan, Nullable&lt;TimeSpan&gt; | Time |
| [MapToPgIntervalAttribute](/api-pg/ArgentSea.Pg..html) | | TimeSpan, Nullable&lt;TimeSpan&gt; | Interval |
| [MapToPgTimeTzAttribute](/api-pg/ArgentSea.Pg.MapToPgTimeTzAttribute.html) | | TimeSpan, DateTimeOffset, Nullable&lt;TimeSpan&gt;, Nullable&lt;DateTimeOffset&gt; | TimeTz |
| [MapToPgArrayAttribute](/api-pg/ArgentSea.Pg.MapToPgArrayAttribute.html) | | Array | Array |
| [MapToPgByteaAttribute](/api-pg/ArgentSea.Pg.MapToPgByteaAttribute.html) | length | byte[] | Bytea |
| [MapToPgArrayAttribute](/api-pg/ArgentSea.Pg.MapToPgArrayAttribute.html) | NpgsqlType | T[]⁴ | Array |
| [MapToPgHstoreAttribute](/api-pg/ArgentSea.Pg.MapToPgHstoreAttribute.html) | length | IDictionary&lt;string, string&gt; | Hstore |
| [MapToPgUuidAttribute](/api-pg/ArgentSea.Pg.MapToPgUuidAttribute.html) | | Guid, Nullable&lt;Guid&gt; | UniqueIdentifier |

¹ The Enum name is saved as string.

² The Enum value is saved based on its underlying numeric value. The Enum integer *base type* (int, short, byte, etc.) must match the database type.

³ This data type is not intended for general use.

⁴ The Npgsql type is used to create parameters; the Property type is used to read them.

***

### Required

Finally, the the data attributes have an optional `required` (sic) parameter. If a database field is DbNull, the Mapper will normally set the corresponding property to null. However, the missing value may represent an entirely absent record. In this case, the correct result is a *null* object, not a valid instance with null/default properties. Setting a property attribute’s `required` argument to True causes the Mapper to return a null object if the property would be null. By default (if not specified), `required` is false.

## Handling Nulls and Empty Types

Because the Mapper is handling database values, there is generally a possibility that the database value is DbNull. How this is converted to a .NET type depends upon the type.

### Strings and Arrays

A .NET string with a value of *null* or a null array will be saved as a DbNull. Empty strings will save as a zero-length string.

### Integers

Integers cannot be null, so the advent of nullable types is a godsend for mapping to database storage. To save or retrieve an integer (byte, Int16, Int32, or Int64) database value from a column that allows null, you should declare a nullable value type.

### Floating Point Numbers

Like integer types, floating point types (Double and Float) can be wrapped in a nullable value. However, ArgentSea also handles *NaN* as a DbNull. If the floating point value is presented as a nullable type, then ArgentSea will save or retrieve NaN; if floating point type is presented, then NaN will be converted to/from a DbNull.

### Guids

Rather like floating point types, Guid.Empty (00000000-0000-0000-0000-000000000000) will be converted to a data DbNull when read from or written to the database. Also like floats, if you need to write an empty Guid value, wrap it in a nullable type.

### Enum Types

.NET enum values can be stored as either numbers or strings. Writing to a text column will automatically save the *name* of the enum; writing to a numeric column saves the *number* value.

> [!WARNING]
> Enums can inherit from several base types (byte, short, int, etc.). If you are saving to a numeric database column, the base type must correctly correspond to the database data type. Enums are Int32 by default.

Nullable Enum types will read or write as a DbNull when the value is null.

### [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html)

These are special types and are discussed in detail in the [sharding](../sharding/sharding.md) section.

### The [MapToModel](/api/ArgentSea.MapToModel.html) attribute

Complex object models may include properties that are objects with their *own* properties, which also need to be mapped to the underlying data.

For example, you might have an Address object that you use for Customers, Vendors, Contacts, Stores, and more. The Vendor class, then, has a property of type Address, and the Customer class has an Address property too. Since the address information is included with the results from the database, the Mapper should map the matching values to the Address object. The `MapToModel` attribute tells the Mapper to do this.

Of course, the property’s type must also have data mapping attributes on the appropriate class properties. The type referenced by a MapToModel attribute can *itself* have a object property with a MapToModel attribute.

In other words, a *Store* object can have a property of type *Address*, which might in turn have a property of type *Coordinates*. If the *Coordinates* type has two properties, each with a `MapToDouble` attribute, the Mapper will be able to map the Latitude and Longitude values to the *Store.Address.Latitude* and *Store.Address.Longitude* fields respectively.

Properties with the [MapToModel](/api/ArgentSea.MapToModel.html) attribute cannot be null, so the root object must be instantiate all of its properties when it is created.

Next: [Mapping Targets](targets.md)