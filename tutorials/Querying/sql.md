
## Creating SQL Queries

Every ArgentSea data access method takes a `Query` argument. ArgentSea offers two implementations:

* `QueryProcedure` - which names a stored procedure (or function). This also sets the DbCommandType to *StoredProcedure*. Of course, the actual SQL implementation is hosted on the server, giving DBAs or SQL developers the ability to revise the implementation, if needed. This is the recommended approach for SQL Server.
* `QueryStatement` - which presents a parameterized SQL statement which is loaded from a file. This sets the DbCommandType to “Text”. This is the recommended approach for PostgreSQL and situations with limited ownership of the database.

ArgentSea recommends creating a single, static class which contains *all* defined procedures or SQL statements as static properties. As large applications evolve, it can become difficult to determine procedures or commands are actually used by the application. Noting Visual Studio’s reference count for each property may help prune dead code.

Here is an example:

```csharp
public static class Queries
{
    // Define query for a SQL Stored Procedure or PostgreSQL function.
    public static QueryProcedure GetSubscriber => new QueryProcedure("ws.ListSubscribers", new[] { "subscriberid" });

    // Define query in a SQL file named “GetSubscriber”. Lazy load and cache.
    private static readonly Lazy<QueryStatement> _getSubscriber = QueryStatement.Create("GetSubscriber", new[] { "subid" });
    public static QueryStatement GetSubscriber => _getSubscriber.Value;

    // add a new static properties for each additional query...
}
```

Now, it is easy to reference the statement batch or procedure like this:

```csharp
return await db.Read.MapReaderAsync<Subscriber>(Queries.GetSubscriber, parameters, cancellation);
```

### QueryProcedures

## [SQL Server](#tab/tabid-sql)

The recommended way to query SQL Server is via stored procedures. This offers performance benefits, the advantage of consolidating SQL in a single repository (SQL Server itself), and the ability to tinker with SQL without changing application code. The defined inputs and output of the stored procedure becomes the defined “interface” to the underlying data.

A `QueryProcedure` class requires a stored procedure name in its constructor. Optionally, you can add parameter names. The advantage of providing parameters names is described below.

If you are referencing a database where creating stored procedures would be problematic (say, you don’t control the target database), then the `Statement` class allows SQL strings to be used instead.

## [PostgreSQL](#tab/tabid-pg)

> [!CAUTION]
> PostgreSQL clients are generally advised to use the `QueryStatement` class, rather than `QueryProcedure`.

You can use the `QueryProcedure` type to call a PostgreSQL function. The longstanding behavior of the [Npgsql](http://www.npgsql.org/) provider is to implicitly change the query to “SELECT * FROM &lt;functionName&gt;”. Consequently, this class can only invoke *functions*, not stored procedures. To invoke stored procedures, use “CALL &lt;sprocName&gt;” in the `QueryStatement` class instead.

A `QueryProcedure` class requires the function name in its constructor. Optionally, you can add parameter names. The advantage of providing parameters names is described below.

***

### QueryStatements

A `QueryStatement` class loads a text file containing SQL commands. The file contains the batch of SQL commands to execute. This helps avoid immutable SQL that is compiled within the application.

The default steps to implement this are:

#### 1. Create a project folder called “SQL”

This is the subdirectory folder from which ArgentSea will load the SQL batch files. The default “SQL” folder name can be changed by setting the static QueryStatement.Folder` property. This should be done during startup.

```csharp
public void ConfigureServices(IServiceCollection services)
{
    QueryStatement.Folder = Configuration["sqlFolder"];
    QueryStatement.Extension = "psql";
    ...
}
```

Alternatively, the `QueryStatement.Create()` factory method overloads allow you specify a folder and extension or absolute path and filename.

#### 2. In the SQL folder, add a new file for each statement batch

The file name should be the same name as the property in your `Queries` class.

## [SQL Server](#tab/tabid-sql)

The default extension for each file is “.sql”. This will be appended to the name you specify. In other words, by default ArgentSea uses “&lt;AppFolder&gt;\SQL\&lt;name&gt;.sql” to locate SQL files.

As with the default folder, you can change the default extension by setting the static `QueryStatement.Extension` property, or by using one of the `Create` overloads.

## [PostgreSQL](#tab/tabid-pg)

When PostgreSQL services are loaded, via the `AddPgServices` call in the `ConfigureServices` method, the default extension for SQL files is set to “psql”. This will be appended to the name you specify. In other words, by default ArgentSea uses “&lt;AppFolder&gt;\SQL\&lt;name&gt;.psql” to locate SQL files.

As with the default folder, you can change the default extension by setting the static `QueryStatement.Extension` property, or by using one of the `Create` overloads. Be sure to set this value, *after* the `AddPgServices` call, but before a query is invoked.

As with the default folder, you can change the default extension by setting the static `QueryStatement.Extension` property, or by using one of the `Create` overloads.

***

### 3. Ensure the SQL files are deployed

In your IDE (i.e. Visual Studio), access the file’s *Properties* pane, then:

* Ensure the *Build Action* attribute is “None”
* Set the *Copy to Output Directory* attribute to “Copy if newer” for each SQL file.

This will ensure that the files are copied to the output/publish directory.

#### 4. Add a corresponding property to the Queries static class

The `QueryStatement` class does not have a default constructor; use the `Create` factory method instead. The `Create` factory method returns a `Lazy<QueryStatement>`. This allows the SQL file to be loaded only when first requested, and then cached for all subsequent requests.

Each static method takes only two lines:

```csharp
private static readonly Lazy<QueryStatement> _getSubscriber = QueryStatement.Create("ListSubscribers");
public static QueryStatement GetSubscriber => _getSubscriber.Value;
```

The create method overloads allow you to explicitly set a folder and extension, or an absolute path and filename.

```csharp
private static readonly Lazy<QueryStatement> _getSubscriber = QueryStatement.Create("ListSubscribers", null, "SqlQueries", "qry");
public static QueryStatement GetSubscriber => _getSubscriber.Value;
```

## Parameter Names

Both the `QueryProcedure` constructor and  the `QueryStatement` factory method take an optional string array of parameter names. If not provided, the Mapper will set/write/read *all* the parameters defined on the Model class; otherwise, only those matching the listed parameter names will be set.

For example, INSERT queries would be unlikely to expect a “LastModified” parameter, but UPDATE queries often do. Unless the appropriate parameters are listed with the Query definition, the ArgentSea Mapper will create parameters from all properties with a corresponding attribute. Extra or missing parameters will generate a query error.

Another scenario where this is helpful is the the “ShardId” parameter. Because this can be implicitly handled by ArgentSea Mapper, there can be ambiguity and inconsistency as to whether or not a shard id parameter should be created. The parameter list helps resolve this.

## [SQL Server](#tab/tabid-sql)

SQL parameter names are always implicitly normalized to include a “@” prefix, so parameters names should match even if the Model’s property attribute does not include the “@” sign. Casing does matter though.

## [PostgreSQL](#tab/tabid-pg)

Parameter names much match, including casing, kana type, etc. Parameter names should generally be consistently lowercase. Future versions of ArgentSea may automatically convert data object names to lowercase.

***

Next: [Coding Queries with ArgentSea](sequence.md)

