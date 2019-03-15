# Mapper Code

Because the work of the Mapper is compiled on the fly, its operation can seem opaque. When logging in “Debug” mode, it will log the code it is creating. For reference, it is also presented here, with an explanation of its logic. This is just an example of the mapping logic, the actual code generated from a different set of metadata attributes will naturally be different.

## Creating and Setting In Parameters

## Creating Out Parameters

## Reading Out Parameters

## Data Reader Queries

Queries involving the data reader are broken into three parts, two of which use code created by expression trees.

The first step captures the ordinal position of every column. This code is executed once per query, at the beginning of the retrieval for each result set. This is a performance optimization, since referencing a column name requires a lookup that should not be repeated on every row.

This means that a change in column order — or additional columns — will not break the code logic. Consequently, the same model class can be used for different queries, each of which may not return exactly the same results. An expected column that is not found will be logged, but subsequently ignored (the type information passed to the GetFieldOrdinal method is for logging).

```csharp
public int[] GetOrdinals(DbDataReader rdr, ILogger logger)
{
    return new[] {
        GetFieldOrdinal(rdr, "customerid", "System.Nullable`1[System.Int32]", logger),
        GetFieldOrdinal(rdr, "locationid", "System.Nullable`1[System.Int16]", logger),
        GetFieldOrdinal(rdr, "locationtypeid", "QuickStart2.Pg.Models.LocationModel+LocationType", logger),
        GetFieldOrdinal(rdr, "streetaddress", "System.String", logger),
        GetFieldOrdinal(rdr, "locality", "System.String", logger),
        GetFieldOrdinal(rdr, "region", "System.String", logger),
        GetFieldOrdinal(rdr, "postalcode", "System.String", logger),
        GetFieldOrdinal(rdr, "iso3166", "System.String", logger),
        GetFieldOrdinal(rdr, "latitude", "System.Double", logger),
        GetFieldOrdinal(rdr, "longitude", "System.Double", logger)
    };
}
```

Next, the data reader handling code iterates through the rows in the result set. Moving through the data reader results is done in standard .NET code, without expression trees. For each new row, the system calls a second delegate, which was also generated and compiled based upon the attribute metadata. The row-handing delegate assesses the current row and returns a model with properties set to data values.

```csharp
public LocationModel ReadData(short shardId, int[] ordinals, DbDataReader rdr, ILogger logger)
{
    var model = new LocationModel();
    var ordinal = ordinals[0];
    short? keyShardId = shardId;
    int? keyRecordId;
    int? keyChildId;
    logger.TraceGetOutMapperProperty("Key_RecordId");

    if (ordinal != -1)
    {
        if (rdr.IsDBNull(ordinal))
        {
            keyRecordId = null;
        }
        else
        {
            keyRecordId = (int?)rdr.GetFieldValue<int>(ordinal);
        }
    }
    logger.TraceGetOutMapperProperty("Key_ChildId");
    ordinal = ordinals[1];
    if (ordinal != -1)
    {
        if (rdr.IsDBNull(ordinal))
        {
            keyChildId = null;
        }
        else
        {
            keyChildId = (int?)rdr.GetFieldValue<short>(ordinal);
        }
    }
    if (keyShardId != null && keyRecordId != null && keyChildId != null)
    {
        model.Key = new ShardChild<short, int, short>('L', keyShardId.Value, keyRecordId.Value, keyChildId.Value);
    }
    else
    {
        model.Key = ShardChild<short, int, short>().Empty;
    }
    logger.TraceRdrMapperProperty("Type");
    ordinal = ordinals[2];
    if (ordinal != -1)
    {
        model.Type = (LocationType)rdr.GetFieldValue<short>(ordinal);
    }
    logger.TraceRdrMapperProperty("StreetAddress");
    ordinal = ordinals[3];
    if (ordinal != -1)
    {
        if (rdr.IsDBNull(ordinal))
        {
            model.StreetAddress = null;
        }
        else
        {
            model.StreetAddress = rdr.GetString(ordinal);
        }
    };
    logger.TraceRdrMapperProperty("Locality");
    ordinal = ordinals[4];
    if (ordinal != -1)
    {
        if (rdr.IsDBNull(ordinal))
        {
            model.Locality = null;
        }
        else
        {
            model.Locality = rdr.GetString(ordinal);
        }
    }
    logger.TraceRdrMapperProperty("Region");
    ordinal = ordinals[5];
    if (ordinal != -1)
    {
        if (rdr.IsDBNull(ordinal))
        {
            model.Region = null;
        }
        else
        {
            model.Region = rdr.GetString(ordinal);
        }
    };
    logger.TraceRdrMapperProperty("PostalCode");
    ordinal = ordinals[6];
    if (ordinal != -1)
    {
        if (rdr.IsDBNull(ordinal))
        {
            model.PostalCode = null;
        }
        else
        {
            model.PostalCode = rdr.GetString(ordinal);
        }
    };
    logger.TraceRdrMapperProperty("Iso3166");
    ordinal = ordinals[7];
    if (ordinal != -1)
    {
        if (rdr.IsDBNull(ordinal))
        {
            model.Iso3166 = null;
        }
        else
        {
            model.Iso3166 = rdr.GetString(ordinal);
        }
    }
    logger.TraceRdrMapperProperty("Latitude");
    ordinal = ordinals[8];
    if (ordinal != -1)
    {
        if (rdr.IsDBNull(ordinal))
        {
            (model.Coordinates).Latitude = double.NaN;
        }
        else
        {
            (model.Coordinates).Latitude = rdr.GetFieldValue<double>(ordinal);
        }
    };
    logger.TraceRdrMapperProperty("Longitude");
    ordinal = ordinals[9];
    if (ordinal != -1)
    {
        if (rdr.IsDBNull(ordinal))
        {
            (model.Coordinates).Longitude = double.NaN;
        }
        else
        {
            (model.Coordinates).Longitude = rdr.GetFieldValue<double>(ordinal)
        }
    };
    return model;
}
```

In this example, the ShardId is obtained from the connection’s property. If the `MapToShardChild` attribute had specified a ShardId column, the code would instead obtained a value from a database column.

## Create a Complex Result

Some of the Mapper’s methods allow you to specify multiple generic arguments. These specify both the base return type and the types for various List properties. This complex model construction is broken down into component steps:

* Create a base model object, either from a single-record data reader result or from output parameters.
* Get one or more list results from subsequent data reader results.
* Assign the list results to the corresponding model properties.

ArgentSea uses reflection to determine which assignable properties match the expected list types, then it builds and compiles a delegate that performs the assignment. This avoids reflection overhead in future cases.

In this example, the Customer model has two list properties: Locations and Contacts. The generated code is straightforward:

```csharp
public CustomerModel CreateComplexModel(
    string queryName,
    DbDataReader,
    IList<CustomerModel> rstResult0,
    IList<LocationModel> rstResult1,
    IList<ContactModel> rstResult2,
    ILogger logger)
{
    var model = AssignRootToResult<TModel>(queryName, rstResult0, logger);
    if (model == null)
    {
        return null;
    }
    model.Locations = rstResult1;
    model.Contacts = rstResult2;
    return model;
}
```

## Loading Multiple Records
