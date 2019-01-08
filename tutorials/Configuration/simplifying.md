# Simplifying Your Data Access Code

As per standard practice in .NET Core, any data repository class can use the ArgentSea data access component by having the service injected into its constructor.

For example, requesting the databases collection in your data access class is straightforward:

## [SQL Server](#tab/tabid-sql)

```csharp
public class MyDataAccessStore
{
  public MyDataAccessStore(SqlDatabases dbs, ILogger<MyDataAccessStore> logger)
  {
     _dbs = dbs;  // capturing injected SqlDatabases collection
    ...
```

The injected data access component allows the class to access any database in the SqlDatabases connection. This means that you would need to specify the collection key to access a particular database or shard set.

In most cases, a class will only access a *single* data source, so, to simplify the data access code, you can instead store only the relevant connection instance:

```csharp
public class MyDataAccessStore
{
    private readonly SqlDatabases.DataConnection _data;

    public MyDataAccessStore(SqlDatabases dbs, ILogger<MyDataAccessStore> logger)
    {
      _data = dbs["MyConnectionKey"];  // capturing relevant database
      ...
```

Setup this way, subsequent calls to the SQL database can be on methods directly on the `_data` object. Now, calls within the class do not need to specify a key.

## [PostgreSQL](#tab/tabid-pg)

```csharp
public class MyDataAccessStore
{
  private readonly PgDatabases  _dbs;

  public MyDataAccessStore(PgDatabases dbs, ILogger<MyDataAccessStore> logger)
  {
     _dbs = dbs;   // capturing injected PgDatabases collection
    ...

```

The injected data access component allows the class to access any database in the PgDatabases connection. This means that you would need to specify the collection key to access a particular database or shard set.

In most cases, a class will only access a *single* data source, so, to simplify the data access code, you can instead store only the relevant connection instance:

```csharp
public class MyDataAccessStore
{
  private readonly PgDatabases.DataConnection _data;

  public MyDataAccessStore(PgDatabases dbs, ILogger<MyDataAccessStore> logger)
  {
    _data = dbs["MyConnectionName"]; // capturing relevant database
    ...
```

Setup this way, calls to the SQL database (or shard set) can be on methods directly on the `_data` object. Now, calls within the class do not need to specify a key.

***

## Simplifying The ShardId Generic Type

The `ShardSets` object has even more need of simplification. As with Databases, a single class typically does not need to access multiple ShardSets, so one can follow the same approach as with the Databases example to reference only the relevant ShardSet within your class.

The other ShardSet complexity is the need to repeatedly declare the *ShardId* type. Within your project, this will always be the same value, but programming against the ArgentSea library directly means declaring the type over and over again.

There are two solutions to this:

* Use the `using` statement to alias the ShardSet declaration.
* Declare classes in your project which inherit from ShardSet, ShardKey, and ShardChild, but with the generic defined. Use these classes in your project.

### Using “using”

As an example of the first approach, to simplify calling a ShardSet *within a single file*, simply add these `using` statements:

## [SQL Server](#tab/tabid-sql)

```csharp
using ShardSets = ArgentSea.Sql.SqlShardSets<byte>;
// and/or
using ShardSet = ArgentSea.Sql.SqlShardSets<byte>.ShardDataSet;
```

This example assumes a ShardId type of *byte*; replace this as appropriate.

An example of how this might be used in the same class we showed before:

```csharp
using ShardSets = ArgentSea.Sql.SqlShardSets<byte>;
using ShardSet = ArgentSea.Sql.SqlShardSets<byte>.ShardDataSet;

public class MyDataAccessStore
{
  private readonly ShardSet _data;

  public MyDataAccessStore(ShardSets shards, ILogger<MyDataAccessStore> logger)
  {
    _data = shards["MyShardSetName"]; //select relevant shard set
    ...

```

## [PostgreSQL](#tab/tabid-pg)

```csharp
using ShardSets = ArgentSea.Pg.PgShardSets<short>;
// and/or
using ShardSet = ArgentSea.Pg.PgShardSets<short>.ShardDataSet;
```

This example assumes a ShardId type of *short*; replace this as appropriate.

An example of how this might be used in the same class we showed before:

```csharp
using ShardSets = ArgentSea.Pg.PgShardSets<short>;
using ShardSet = ArgentSea.Pg.PgShardSets<short>.ShardDataSet;

public class MyDataAccessStore
{
  private readonly ShardSet _data;

  public MyDataAccessStore(ShardSets shards, ILogger<MyDataAccessStore> logger)
  {
    _data = shards["MyShardSetName"]; //select relevant shard set
    ...

```

***

The downside to the `using` approach is that it works for only the current file.

### Child Classes

By creating a local class that inherits from the ArgentSea generic class, you can simplify the shard set reference throughout your project.

## [SQL Server](#tab/tabid-sql)

These are the classes you might create in your SQL project to simplify the ShardSet within our project.

```csharp
public class MyShardSets : SqlShardSets<byte>
{
  public SqlShardSets(
    IOptions<SqlShardConnectionOptions<TShard>> configOptions,
    IOptions<SqlGlobalPropertiesOptions> globalOptions,
    ILogger<SqlShardSets<TShard>> logger
  ) : base(configOptions, globalOptions, logger)
    {
        //
    }
}

public class MyShardKey : ShardKey<byte>
{
    //
}

public class MyShardChild : ShardChild<byte>
{
    //
}

```

## [PostgreSQL](#tab/tabid-pg)

```csharp
public class MyShardSets : PgShardSets<short>
{
  public SqlShardSets(
    IOptions<PgShardConnectionOptions<TShard>> configOptions,
    IOptions<PgGlobalPropertiesOptions> globalOptions,
    ILogger<PgShardSets<TShard>> logger
  ) : base(configOptions, globalOptions, logger)
    {
        //
    }
}

public class MyShardKey : ShardKey<short>
{
    //
}

public class MyShardChild : ShardChild<short>
{
    //
}

```

***

This approach will be helpful in reducing the number of times the generic *ShardId* type must be specified in your project code.

Next: [Resilience Strategies](resilience.md)
