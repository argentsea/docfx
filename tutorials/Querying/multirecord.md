# Multi-record Saves

One of the best ways to improve performance is to simply reduce the number of server round-trips. When transactions need to save a list of order items, or customer locations, or other related data, the entire data set should be loaded and committed as a batch. Unfortunately, ADO.Net makes this a little more difficult because it doesn’t offer consistency across platforms.

## [SQL Server](#tab/tabid-sql)

Although SQL Server can accept multi-valued parameters via XML or JSON or even parsable strings, the standard, recommended approach is to use Table Valued Parameters. ArgentSea offers a Mapper for TVPs too.

The TVP Mapper accepts a Model object and returns a `SqlDataRecord` build from the mapping attributes. The Table Valued Parameters accepts a collection (list, array, etc.) or the `SqlDataRecord` objects. Like all Mapper processes, the conversion logic is extracted and compiled on the first call, offering native-compiliation performance on subsequent calls.

An implementation, then, simply iterates the list of Model objects, creating the corresponding list of `SqlDataRecords`. The list of records is a assigned to a parameter like any other value. Like this:

```csharp
var contactRecords = new List<SqlDataRecord>();
customer.Contacts.ForEach(contact =>
{
    contactRecords.Add(TvpMapper.ToTvpRecord<ContactModel>(contact, _logger));
});
var prms = new ParameterCollection()
    .AddSqlIntInputParameter("@CustomerId")
    .AddSqlTableValuedParameter("@Contacts", contactRecords);
```

For higher performance with very large data sets, the Batch functionality could allow someone to implement a BatchStep that uses the SqlBulkCopy class. This would be similar to the approach used by the PostgreSQL implementation.

## [PostgreSQL](#tab/tabid-pg)

ArgentSea supports PostgreSQL’s COPY functionality using the NpgsqlBinaryImporter. This is how data sets can be efficiently loaded into PostgreSQL.

ArgentSea’s [Batch](batches.md) functionality is used to load data into PostgreSQL tables or temporary tables, then a SQL statement is used to process and commit these values. The Batch automatically ensures that both steps share the same transaction.

This example creates a batch, then adds a step to push a series of values to a temporary table called “t-locations”. The next step runs a SQL statement that uses both the SQL parameters and the temporary table to save the data.

```csharp
using ArgentSea.Pg;
//...
var customerPrms = new ParameterCollection()
    .AddPgVarcharInputParameter("customername", customer.Name, 255);
var shardBatch = new ShardBatch<short, ShardKey<short, int>>()
    .Add(customer.Locations, "t-locations")
    .Add(Queries.CustomerSave, customerPrms, DataOrigins.Customer, "customerid");
var custKey = await _shardSet.DefaultShard.Write.RunAsync(shardBatch, cancellation);
```

***

Next: [Sharding](../sharding/sharding.md)
