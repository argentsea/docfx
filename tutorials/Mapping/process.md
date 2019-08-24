# Mapper Code

Because the work of the Mapper is compiled on the fly, its operation can seem opaque, especially when you are trying to debug problems. The Visual Studio debugger offers almost no help. Consequently, ArgentSea offers extensive tracing-level logging and also captures a rendering of the generated code as each model is compiled. This should assist in identifying any problems. In the sample applications, the default logger will write to the Visual Studio output window.

> [!NOTE]
> Tracing requires a logging level of “Trace”; the generated code is only logged with the logging level of “Debug” or “Trace”.

If you are curious about the mapping logic, examples are presented here which is based upon a sample Model. Or course, the actual code generated from a different Model or different set of metadata attributes will be different. The general logic, however, will be largely intact.

## Creating and Setting Input Parameters

Some overloads of the CreateInputParameters methods allow you to specify parameters that should not be set. Any parameters *already* set are added to this list, so that this delegate does not attempt to add them again. This allows you to, say, set an output parameter before setting all the remaining input parameters with this method.

## [SQL Server](#tab/tabid-sql)

```csharp
public void CreateAndSetInputParameter(LocationModel model, DbParameterCollection parameters, HashSet<string> ignoreParameters, ILogger logger)
{
    int? keyRecordId;
    short? keyChildId;

    logger.TraceInMapperProperty("Key");
    if (model.Key != ShardKey<short, int, short>.Empty)
    {
        keyRecordId = model.Key.RecordId;
        keyChildId = model.Key.ChildId;
    }
    else
    {
        keyRecordId = null;
        keyChildId = null;
    }

    if (ExpressionHelpers.DontIgnoreThisParameter("@CustomerId", ignoreParameters))
    {
        parameters.AddSqlIntInputParameter("@CustomerId", keyRecordId)
    }
    if (ExpressionHelpers.DontIgnoreThisParameter("@LocationId", ignoreParameters))
    {
        parameters.AddSqlSmallIntInputParameter("@LocationId", keyChildId)
    }

    logger.TraceInMapperProperty("Type");
    if (ExpressionHelpers.DontIgnoreThisParameter("@LocationTypeId", ignoreParameters))
    {
        parameters.AddSqlTinyIntInputParameter("@LocationTypeId", (byte)model.Type)
    }

    logger.TraceInMapperProperty("StreetAddress");
    if (ExpressionHelpers.DontIgnoreThisParameter("@StreetAddress", ignoreParameters))
    {
        parameters.AddSqlNVarCharInputParameter("@StreetAddress", model.StreetAddress, 255)
    }

    logger.TraceInMapperProperty("Locality");
    if (ExpressionHelpers.DontIgnoreThisParameter("@Locality", ignoreParameters))
    {
        parameters.AddSqlNVarCharInputParameter("@Locality", model.Locality, 100)
    }

    logger.TraceInMapperProperty("Region");
    if (ExpressionHelpers.DontIgnoreThisParameter("@Region", ignoreParameters))
    {
        parameters.AddSqlNVarCharInputParameter("@Region", model.Region, 100)
    }

    logger.TraceInMapperProperty("PostalCode");
    if (ExpressionHelpers.DontIgnoreThisParameter("@PostalCode", ignoreParameters))
    {
        parameters.AddSqlNVarCharInputParameter("@PostalCode", model.PostalCode, 25)
    }

    logger.TraceInMapperProperty("Iso3166");
    if (ExpressionHelpers.DontIgnoreThisParameter("@Iso3166", ignoreParameters))
    {
        parameters.AddSqlNCharInputParameter("@Iso3166", model.Iso3166, 2)
    }

    logger.TraceInMapperProperty("Latitude");
    if (ExpressionHelpers.DontIgnoreThisParameter("@Latitude", ignoreParameters))
    {
        parameters.AddSqlFloatInputParameter("@Latitude", model.Latitude)
    }

    logger.TraceInMapperProperty("Longitude");
    if (ExpressionHelpers.DontIgnoreThisParameter("@Longitude", ignoreParameters))
    {
        parameters.AddSqlFloatInputParameter("@Longitude", model.Longitude)
    }
}

```

## [PostgreSQL](#tab/tabid-pg)

```csharp
public void CreateAndSetInputParameter(LocationModel model, DbParameterCollection parameters, HashSet<string> ignoreParameters, ILogger logger)
{
    int? keyRecordId;
    short? keyChildId;

    logger.TraceInMapperProperty("Key");
    if (model.Key != ShardKey<short, int, short>.Empty)
    {
        keyRecordId = model.Key.RecordId;
        keyChildId = model.Key.ChildId;
    }
    else
    {
        keyRecordId = null;
        keyChildId = null;
    }

    if (ExpressionHelpers.DontIgnoreThisParameter("customerid", ignoreParameters))
    {
        parameters.AddPgIntegerInputParameter("customerid", keyRecordId)
    }
    if (ExpressionHelpers.DontIgnoreThisParameter("locationid", ignoreParameters))
    {
        parameters.AddPgSmallintInputParameter("locationid", keyChildId)
    }

    logger.TraceInMapperProperty("Type");
    if (ExpressionHelpers.DontIgnoreThisParameter("locationtypeid", ignoreParameters))
    {
        parameters.AddPgSmallintInputParameter("locationtypeid", (short)model.Type)
    }

    logger.TraceInMapperProperty("StreetAddress");
    if (ExpressionHelpers.DontIgnoreThisParameter("streetaddress", ignoreParameters))
    {
        parameters.AddPgVarcharInputParameter("streetaddress", model.StreetAddress, 255)
    }

    logger.TraceInMapperProperty("Locality");
    if (ExpressionHelpers.DontIgnoreThisParameter("locality", ignoreParameters))
    {
        parameters.AddPgVarcharInputParameter("locality", model.Locality, 100)
    }

    logger.TraceInMapperProperty("Region");
    if (ExpressionHelpers.DontIgnoreThisParameter("region", ignoreParameters))
    {
        parameters.AddPgVarcharInputParameter("region", model.Region, 100)
    }

    logger.TraceInMapperProperty("PostalCode");
    if (ExpressionHelpers.DontIgnoreThisParameter("postalcode", ignoreParameters))
    {
        parameters.AddPgVarcharInputParameter("postalcode", model.PostalCode, 25)
    }

    logger.TraceInMapperProperty("Iso3166");
    if (ExpressionHelpers.DontIgnoreThisParameter("iso3166", ignoreParameters))
    {
        parameters.AddPgCharInputParameter("iso3166", model.Iso3166, 2)
    }

    logger.TraceInMapperProperty("Latitude");
    if (ExpressionHelpers.DontIgnoreThisParameter("latitude", ignoreParameters))
    {
        parameters.AddPgDoubleInputParameter("latitude", model.Latitude)
    }

    logger.TraceInMapperProperty("Longitude");
    if (ExpressionHelpers.DontIgnoreThisParameter("longitude", ignoreParameters))
    {
        parameters.AddPgDoubleInputParameter("longitude", model.Longitude)
    }
}

```

***

## Creating Out Parameters

The code generated to set output parameters is very straightforward. This example is SQL Server-specific, but if you chose to use output parameters on another platform, the logic would be the same.

```csharp
public void CreateOutParameters (DbParameterCollection parameters, HashSet<string> ignoreParameters, ILogger logger)
{
    logger.TraceSetOutMapperProperty("Key");
    if (ExpressionHelpers.DontIgnoreThisParameter("@CustomerId", ignoreParameters))
    {
        parameters.AddSqlIntOutputParameter("@CustomerId");
    }
    if (ExpressionHelpers.DontIgnoreThisParameter("@LocationId", ignoreParameters))
    {
        parameters.AddSqlSmallIntOutputParameter("@LocationId");
    }

    logger.TraceSetOutMapperProperty("Type");
    if (ExpressionHelpers.DontIgnoreThisParameter("@LocationTypeId", ignoreParameters))
    {
        parameters.AddSqlIntOutputParameter("@LocationTypeId");
    }

    logger.TraceSetOutMapperProperty("StreetAddress");
    if (ExpressionHelpers.DontIgnoreThisParameter("@StreetAddress", ignoreParameters))
    {
        parameters.AddSqlNVarCharOutputParameter("@StreetAddress", 255);
    }

    logger.TraceSetOutMapperProperty("Locality");
    if (ExpressionHelpers.DontIgnoreThisParameter("@Locality", ignoreParameters))
    {
        parameters.AddSqlNVarCharOutputParameter("@Locality", 100);
    }

    logger.TraceSetOutMapperProperty("Region");
    if (ExpressionHelpers.DontIgnoreThisParameter("@Region", ignoreParameters))
    {
        parameters.AddSqlNVarCharOutputParameter("@Region", 100);
    }

    logger.TraceSetOutMapperProperty("PostalCode");
    if (ExpressionHelpers.DontIgnoreThisParameter("@PostalCode", ignoreParameters))
    {
        parameters.AddSqlNVarCharOutputParameter("@PostalCode", 25);
    }

    logger.TraceSetOutMapperProperty("Iso3166");
    if (ExpressionHelpers.DontIgnoreThisParameter("@Iso3166", ignoreParameters))
    {
        parameters.AddSqlNCharOutputParameter("@Iso3166", 2);
    }

    logger.TraceSetOutMapperProperty("Latitude");
    if (ExpressionHelpers.DontIgnoreThisParameter("@Latitude", ignoreParameters))
    {
        parameters.AddSqlFloatOutputParameter("@Latitude");
    }

    logger.TraceSetOutMapperProperty("Longitude");
    if (ExpressionHelpers.DontIgnoreThisParameter("@Longitude", ignoreParameters))
    {
        parameters.AddSqlFloatOutputParameter("@Longitude");
    }
}
```

## Reading Out Parameters

This example evaluates only a few output parameters: Customer key, CustomerType, and Customer Name. The Key and Name properties are required, so we consider database Null values to represent no record.

Notably, this example handles a ShardKey value. If the ShardId parameter had been specified, it would have been captured and evaluated like the RecordId parameter; if either were null then the ShardKey value would be empty. However, since no ShardId parameter was specified, the code uses the shardId of the current shard, which is provided to the procedure as an argument.

```csharp
public Customer ReadOutParameters(short shardId, DbParameterCollection parameters, ILogger logger)
{
    var model = new CustomerModel();
    DbParameter prm;
    short? keyShardId;
    int? keyRecordId;

    logger.TraceGetOutMapperProperty("Key_RecordId");
    prm = ExpressionHelpers.GetParameter(parameters, "CustomerId");
    if (prm != null)
    {
        keyShardId = prm.GetNullableInteger();
    }
    else
    {
        logger.SqlParameterNotFound("@CustomerId", typeof(int?));
    };
    if (ExpressionHelpers.IsRequiredParameterDbNull(prm, "CustomerModel", "@CustomerId", logger))
    {
        return null;
    }

    logger.TraceGetOutMapperProperty("Name");
    prm = ExpressionHelpers.GetParameter(parameters, "@Name");
    if (ExpressionHelpers.IsRequiredParameterDbNull(prm, "CustomerModel", "@Name", logger))
    {
        return null;
    }
    if (prm != null)
    {
        model.Name = prm.GetString();
    }

    logger.TraceGetOutMapperProperty("Type");
    prm = ExpressionHelpers.GetParameter(parameters, "@CustomerTypeId");
    if (prm != null)
    {
        model.Type = (CustomerType)prm.GetShort();
    }
    else
    {
        logger.SqlParameterNotFound("@CustomerTypeId", typeof(CustomerType));
    }

    keyShardId = shardId;
    if (keyShardId != null && keyRecordId != null)
    {
        model.Key = ShardKey<short, int>('C', keyShardId.Value, keyRecordId.Value);
    }
    else
    {
        model.Key = ShardKey<short, int>.Empty;
    };
}
```

## Data Reader Queries

Queries involving the data reader follows these steps for each result set:

* Use generated code to build an array of column ordinal positions based on the expected column names.
* Create a list result, then loop through each record in the result set.
* For each record in the result set, call a generated function which returns a new Model object with property values set to the corresponding columns values.
* Return a list of the new Model objects.

The initial capture of ordinal positions is a performance optimization, since the column positions will not change and accessing a column by name always requires a ordinal lookup. This also means that a change in column order — or additional columns — will not break the code logic. Consequently, the same model class can be used for different queries, each of which may not return exactly the same results. An expected column that is not found will be logged (at “Debug” logging level), but subsequently ignored. (If you are curious, the type information passed to the GetFieldOrdinal method is for logging).

The code looks like this:

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

Reading through the DataReader rows does not require Mapper generated code. 

For each new row, the system calls a second delegate, which was also generated and compiled based upon the attribute metadata. The row-handing delegate assesses the current row and returns a model with properties set to data values. 

Here are a few notes about the code:

* Ordinal values of -1 indicate that the column was not found, in which case the property will be ignored.
* The Trace logging commands might be useful in determining the Mapper’s last operation before an unexpected failure.
* In this example, the ShardId is obtained from the connection’s property. If the `MapToShardKey` attribute had specified a ShardId column, the code would instead be obtained a value from a database column.


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
        model.Key = new ShardKey<short, int, short>('L', keyShardId.Value, keyRecordId.Value, keyChildId.Value);
    }
    else
    {
        model.Key = ShardKey<short, int, short>().Empty;
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


## Create a Complex Result

Some of the Mapper’s methods allow you to specify multiple generic arguments. These specify both the base return type and the types for various List properties. This complex model construction is broken down into component steps:

* Create a base model object, either from a single-record data reader result, from output parameters, or (failing those options) by creating a new instance.
* Match the type of each List parameter to any Model property of the same type, and assign the property to the List parameter value.
* Repeat the List matching until all parameters have been assigned.

> [!WARNING]
> Note that due to this logic, the Mapper cannot manage multiple List properties of the same type.

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

## [SQL Server](#tab/tabid-sql)

SQL Server used Table Valued Parameters to send multple records to a stored procedure. The logic that iterates over the Model list does not require generated code. However, for each item in the Model list, the system calls a generated delegate to convert the Model instance to a `SqlDataRecord`. 

```csharp
public SqlDataRecord SetTvpRow(LocationModel model, IList<string> columnList, ILogger logger)
{
    int? keyRecordId;
    short? keyChildId;
    var fields = new SqlMetaData[10] {
            new SqlMetaData("CustomerId", SqlDbType.Int),
            new SqlMetaData("LocationId", SqlDbType.SmallInt),
            new SqlMetaData("LocationTypeId", SqlDbType.TinyInt),
            new SqlMetaData("StreetAddress", SqlDbType.NVarChar, 255L),
            new SqlMetaData("Locality", SqlDbType.NVarChar, 100L),
            new SqlMetaData("Region", SqlDbType.NVarChar, 100L),
            new SqlMetaData("PostalCode", SqlDbType.NVarChar, 25L),
            new SqlMetaData("Iso3166", SqlDbType.NChar, 2L),
            new SqlMetaData("Latitude", SqlDbType.Float),
            new SqlMetaData("Longitude", SqlDbType.Float)
        };
    var result = new SqlDataRecord(GetRecordDataFields(fields, columnList));

    logger.TraceTvpMapperProperty("Key");
    if (model.Key != ShardKey<short, int, short>.Empty)
    {
        keyRecordId = model.Key.RecordId;
    }
    else
    {
        keyRecordId = null;
    }
    if (TvpExpressionHelpers.IncludeThisColumn("CustomerId", columnList))
    {
        if (keyRecordId.HasValue)
        {
            result.SetInt32(TvpExpressionHelpers.GetOrdinal(0, "CustomerId", columnList), keyRecordId.Value);
        }
        else
        {
            result.SetDBNull(TvpExpressionHelpers.GetOrdinal(0, "CustomerId", columnList));
        }
    }
    if (model.Key != ArgentSea.ShardKey<short, int, short>.Empty)
    {
        KeyChildId = model.Key.ChildId;
    }
    else
    {
        keyChildId = null;
    }
    if (TvpExpressionHelpers.IncludeThisColumn("LocationId", columnList))
    {
        if (keyChildId.HasValue))
        {
            result.SetInt16(TvpExpressionHelpers.GetOrdinal(1, "LocationId", columnList), keyChildId.Value);
        }
        else
        {
            result.SetDBNull(TvpExpressionHelpers.GetOrdinal(1, "LocationId", columnList));
        }
    }

    logger.TraceTvpMapperProperty("Type");
    if (TvpExpressionHelpers.IncludeThisColumn("LocationTypeId", columnList))
    {
        result.SetByte(TvpExpressionHelpers.GetOrdinal(2, "LocationTypeId", columnList), (byte)model.Type);
    }

    logger.TraceTvpMapperProperty("StreetAddress");
    if (TvpExpressionHelpers.IncludeThisColumn("StreetAddress", columnList))
    {
        if (model.StreetAddress == null)
        {
            result.SetDBNull(TvpExpressionHelpers.GetOrdinal(3, "StreetAddress", columnList));
        }
        else
        {
            result.SetString(TvpExpressionHelpers.GetOrdinal(3, "StreetAddress", columnList), model.StreetAddress);
        }
    }


    logger.TraceTvpMapperProperty("Locality");
    if (TvpExpressionHelpers.IncludeThisColumn("Locality", columnList))
    {
        if (model.Locality == null)
        {
            result.SetDBNull(TvpExpressionHelpers.GetOrdinal(4, "Locality", columnList));
        }
        else
        {
            result.SetString(TvpExpressionHelpers.GetOrdinal(4, "Locality", columnList), model.Locality);
        }
    }

    logger.TraceTvpMapperProperty("Region");
    if (TvpExpressionHelpers.IncludeThisColumn("Region", columnList))
    {
        if (model.Region == null)
        {
            result.SetDBNull(TvpExpressionHelpers.GetOrdinal(5, "Region", columnList));
        }
        else
        {
            result.SetString(TvpExpressionHelpers.GetOrdinal(5, "Region", columnList), model.Region);
        }
    }

    logger.TraceTvpMapperProperty("PostalCode");
    if (TvpExpressionHelpers.IncludeThisColumn("PostalCode", columnList))
    {
        if (model.PostalCode == null)
        {
            result.SetDBNull(TvpExpressionHelpers.GetOrdinal(6, "PostalCode", columnList));
        }
        else
        {
            result.SetString(TvpExpressionHelpers.GetOrdinal(6, "PostalCode", columnList), model.PostalCode);
        }
    }

    logger.TraceTvpMapperProperty("Iso3166");
    if (TvpExpressionHelpers.IncludeThisColumn("Iso3166", columnList))
    {
        if (model.Iso3166 == null)
        {
            result.SetDBNull(TvpExpressionHelpers.GetOrdinal(7, "Iso3166", columnList));
        }
        else
        {
            result.SetString(TvpExpressionHelpers.GetOrdinal(7, "Iso3166", columnList), model.Iso3166);
        }
    }

    logger.TraceTvpMapperProperty("Latitude");
    if (TvpExpressionHelpers.IncludeThisColumn("Latitude", columnList))
    {
        if (Double.IsNaN(model.Coordinates.Latitude))
        {
            result.SetDBNull(TvpExpressionHelpers.GetOrdinal(8, "Latitude", columnList));
        }
        else
        {
            result.SetDouble(TvpExpressionHelpers.GetOrdinal(8, "Latitude", columnList), model.Coordinates.Latitude);
        }
    }

    logger.TraceTvpMapperProperty("Longitude");
    if (TvpExpressionHelpers.IncludeThisColumn("Longitude", columnList))
    {
        if (Double.IsNaN(model.Coordinates.Longitude))
        {
            result.SetDBNull(TvpExpressionHelpers.GetOrdinal(9, "Longitude", columnList));
        }
        else
        {
            result.SetDouble(TvpExpressionHelpers.GetOrdinal(9, "Longitude", columnList), model.Coordinates.Longitude);
        }
    }
    return result;
}
```

## [PostgreSQL](#tab/tabid-pg)

PostgreSQL uses the Npgsql COPY functionality to send multple records to the database. The COPY function may use an new, existing, or temporary table. If a period (schema specifier) exists in the table name, then a “CREATE TABLE IF NOT EXISTS” command is used, otherwise the Mapper uses “CREATE TEMP TABLE”.

```SQL
CREATE TEMP TABLE  "temp_locations" (
    "locationid" smallint NULL,
    "locationtypeid" smallint NULL,
    "streetaddress" varchar(255) NULL,
    "locality" varchar(100) NULL,
    "region" varchar(100) NULL,
    "postalcode" varchar(25) NULL,
    "iso3166" varchar(2) NULL,
    "latitude" double precision NULL,
    "longitude" double precision NULL); 

```

Once the table is created (if necessary), the Mapper sends the SQL command to initiate the PostgreSQL COPY command:

```SQL
COPY "temp_locations" ("locationid","locationtypeid","streetaddress","locality","region","postalcode","iso3166","latitude","longitude") FROM STDIN BINARY;
```

Finally, the Mapper opens a `NpgsqlDataImporter`. For each item in the Model List, the Mapper invokes this generated code:

```csharp
public void SetRow(NpgsqlDataImporter importer, CustomerLocation model)
    importer.StartRow();
    importer.Write(model.Sequence, NpgsqlDbType.Smallint);
    importer.Write(model.Type, NpgsqlDbType.Smallint);
    if (model.StreetAddress == null)
    {
        importer.WriteNull();
    }
    else
    {
        importer.Write(model.StreetAddress, NpgsqlDbType.Varchar);
    }
    if (model.Locality == null)
    {
        importer.WriteNull();
    }
    else
    {
        importer.Write(model.Locality, NpgsqlDbType>(Varchar);
    }
    if (model.Region == null)
    {
        importer.WriteNull();
    }
    else
    {
        importer.Write(model.Region, NpgsqlDbType>(Varchar);
    }
    if (model.PostalCode == null)
    {
        importer.WriteNull();
    }
    else
    {
        importer.Write(model.PostalCode, NpgsqlDbType.Varchar);
    }
    if (model.Iso3166 == null)
    {
        importer.WriteNull();
    }
    else
    {
        importer.Write(model.Iso3166, NpgsqlDbType.Varchar);
    }
    if (Double.IsNaN((model.Coordinates.Latitude))
    {
        importer.WriteNull();
    }
    else
    {
        importer.Write(model.Coordinates.Latitude, NpgsqlDbType.Double)
    };
    if (Double.IsNaN((model.Coordinates.Longitude)
    {
        importer.WriteNull();
    }
    else
    {
        importer.Write(model.Coordinates.Longitude, NpgsqlDbType>(Double);
    }
}
```

Now that the table “temp_locations” has the corresponding records, it only takes a straightforward SQL statement to write these values into the final table (presuming that the COPY function isn’t writing to the final destination). ArgentSea’s Batch functionality allows multiple steps to operation on the same transaction, so temporary tables are not dropped prior to SQL statement execution.

***

Next: [Logging](logging.md)