# Simplifying Your Data Access Code

As per standard practice in .NET Core, any data repository class can use the ArgentSea data access component by having the service injected into its constructor.

For example, requesting the databases collection in your data access class is straightforward:

## [SQL Server](#tab/tabid-sql)

```csharp
public class MyDataAccessStore
{
  private readonly SqlDatabases _dbs;

  public MyDataAccessStore(SqlDatabases dbs, ILogger<MyDataAccessStore> logger)
  {
     _dbs = dbs;  // capturing injected SqlDatabases collection
    ...
```

Likewise, access to the entire collection of shard sites via injection:

```csharp
public class MyDataAccessStore
{
  private readonly SqlShardSets _shardSets;

  public MyDataAccessStore(SqlShardSets shardsets, ILogger<MyDataAccessStore> logger)
  {
     _shardSets = shardSets;  // capturing injected SqlShardSets collection
    ...
```

In most cases, your repository class would not require access to all shard sets or all databases. Capturing the entire collection also requires specifying the collection key for every data access. A class will generally only access a *single* data source, so, to simplify the data access code, you can instead store only the relevant connection instance:

```csharp
public class MyDataAccessStore
{
    private readonly SqlDatabases.DataConnection _data;

    public MyDataAccessStore(SqlDatabases dbs, ILogger<MyDataAccessStore> logger)
    {
      _data = dbs["MyConnectionKey"];  // capturing only the relevant database
      ...
```

Or

```csharp
public class MyDataAccessStore
{
    private readonly SqlShardSets.ShardDataSet _shards;

    public MyDataAccessStore(SqlShardSets shardSets, ILogger<MyDataAccessStore> logger)
    {
      _shards = shardSets["MyShardSetKey"];  // capturing only the relevant shard set
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

Likewise, access to the entire collection of shard sites via injection:

```csharp
public class MyDataAccessStore
{
  private readonly PgShardSets _shardSets;

  public MyDataAccessStore(PgShardSets shardsets, ILogger<MyDataAccessStore> logger)
  {
     _shardSets = shardSets;  // capturing injected SqlShardSets collection
    ...
```

In most cases, your repository class would not require access to all shard sets or all databases. Capturing the entire collection also requires specifying the collection key for every data access. A class will generally only access a *single* data source, so, to simplify the data access code, you can instead store only the relevant connection instance:

```csharp
public class MyDataAccessStore
{
    private readonly PgDatabases.DataConnection _data;

    public MyDataAccessStore(PgDatabases dbs, ILogger<MyDataAccessStore> logger)
    {
      _data = dbs["MyConnectionKey"];  // capturing only the relevant database
      ...
```

Or

```csharp
public class MyDataAccessStore
{
    private readonly PgShardSets.ShardDataSet _shards;

    public MyDataAccessStore(PgShardSets shardSets, ILogger<MyDataAccessStore> logger)
    {
      _shards = shardSets["MyShardSetKey"];  // capturing only the relevant shard set
      ...
```

Setup this way, subsequent calls to the SQL database can be on methods directly on the `_data` object. Now, calls within the class do not need to specify a key.

***

Next: [Data Mapping](../mapping/mapping.md)
