# QuickStart One

This article will step you through a simple setup of ArgentSea for non-sharded data access. This presentation introduces concepts which are further elaborated in the [subsequent article](sharding.md), so this is a good orientation prior that that walkthrough too. 

If you get stuck or have questions, click on one of the links to the in-depth tutorial articles.

## 1. Create a Project (or use an existing one)

The sample QuickStart project is [here](https://github.com/argentsea/quickstarts). You can reproduce the starting point by creating a new ASP.NET API project type. If you prefer to start by creating a new, empty project, ensure that appsettings.json is added.

## 2. Setup your Database

If you are using the QuickStart1 project, a setup SQL script is including in the project; otherwise you can access it on [GitHub](https://github.com/argentsea/quickstarts). You can run the SQL script against the database server to create the sample databases, tables, data, etc.

## [SQL Server](#tab/tabid-sql)

The best way to use ArgentSea is to exclusively use stored procedures, and to enforce least-privileged access by granting the logon *only* EXECUTE permissions.

Going further, you might consider creating distinct schemas, one for SELECT (read) procedures and one for INSERT, UPDATE, DELETE (write) procedures. Create two users and grant each EXECUTE to one or the other schema. Besides adding additional security, this may also help validate scale-out read-only endpoints during development and testing.

## [PostgreSQL](#tab/tabid-pg)

As part of a general approach to lease-privileged access, you might consider creating two database users — one for read access and the other for writes. This approach can also help validate scale-out read-only endpoints during development and testing.

***

> [!CAUTION]
> A password is embedded in the setup script (and corresponds to the credential sections below); you might consider changing this value. However, this is also a low-privileged user, so it is not a high risk if you leave things as they are.

## 3. Add ArgentSea to your project

If you have loaded the QuickStart project, you can skip this step. If you are creating a new project, use NuGet to add ArgentSea to your project by selected the package that corresponds to your database platform:

* For Microsoft SQL Server databases, use **ArgentSea.Sql**
* For PostgreSQL, use **ArgentSea.Pg**

Both packages will automatically include the shared ArgentSea package and any other dependencies. Using both packages in the same project may work but is not a tested scenario.

You can learn more about adding a reference to ArgentSea [here](https://docs.microsoft.com/en-us/nuget/quickstart/install-and-use-a-package-in-visual-studio).

## 4. Define your Login Information

ArgentSea leverages the .NET Core configuration architecture, which means that the configuration information is combined from multiple providers. In this example, we will store most connection information in the *appsettings.json* file, but store the login password securely in a separate store.

> [!TIP]
> Because the new configuration architecture in .NET core allows values to be hosted in multiple places, we can also use environment variables — which can be very useful for managing a release pipeline. In that case, we might store the server or database information there instead of *appsettings.json*. Again, the details are described in the [configuration](/tutorials/configuration/configuration.md) tutorial.

Open the [User Secrets](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets) JSON configuration file by right-clicking on the project node and selecting it from the popup menu. If a User Secrets option does not appear, you may need to add the *Microsoft.Extensions.Configuration.UserSecrets* NuGet package to your project. User Secrets provides a means of keeping passwords out of your source control files.

> [!NOTE]
> Because the QuickStart1 sample application uses User Secrets, those following along at home with the downloaded project will still need to manually copy the credentials to the User Secrets in the sample app.

# [SQL Server](#tab/tabid-sql)

To connect using username and password authentication, add this to your User Secrets:

```json
{
  "SqlDbConnections": [
    {
      "Password": "Pwd123456"
    }
  ]
}
```

Change the values, as appropriate, to represent a valid login.

Configure the *DataSource* and *InitialCatalog* properties in your *appsettings.json* configuration file:

```json
{
  "SqlDbConnections": [
    {
      "DatabaseKey": "MyDatabase",
      "UserName": "webUser",
      "DataSource": "localhost",
      "InitialCatalog": "MyDb"
    }
  ]
}
```

Note that if you are using Windows authentication, you can just specify this in *appsettings.json* and *you don’t need to manage User Secrets*:

```json
{
  "SqlDbConnections": [
    {
      "DatabaseKey": "MyDatabase",
      "WindowsAuth": true,
      "DataSource": "localhost",
      "InitialCatalog": "MyDb"
    }
  ]
}
```

# [PostgreSQL](#tab/tabid-pg)

To connect using username and password authentication, add this to your User Secrets:

```json
{
  "PgDbConnections": [
    {
      "Password": "Pwd123456"
    }
  ]
}
```

Configure the *Host* and *Database* properties in your *appsettings.json* configuration file:

```json
{
  "PgDbConnections": [
    {
      "DatabaseKey": "MyDatabase",
      "Host": "localhost",
      "Database": "MyDb",
      "UserName": "webuser"
    }
  ]
}

```

Note that if you are using Windows authentication, you can just specify this in *appsettings.json* and you don’t need to manage secrets:

```json
{
  "PgDbConnections": [
    {
      "DatabaseKey": "MyDatabase",
      "Host": "localhost",
      "Database": "MyDb",
      "WindowsAuth": true
    }
  ]
}
```

***

> [!CAUTION]
> In a production deployment, the Credential section — entries currently placed in [User Secrets](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets) — should be hosted in [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/), [Azure Key Vault](https://azure.microsoft.com/en-us/services/key-vault/), a secure file share, or some other secure resource.

## 7. Load ArgentSea on Application Start

ArgentSea is an injectable service, so it needs to be registered on application startup.

# [SQL Server](#tab/tabid-sql)

Open your project’s *Startup* class. At the top, there should be the following *using* statement:

```csharp
using ArgentSea.Sql;
```

Then, in the `Startup` class’ `ConfigureServices` method, add:

```csharp
services.AddSqlServices(Configuration);
```

This step creates an injectable *SqlServices* object that we can consume in all of our data access clients.

# [PostgreSQL](#tab/tabid-pg)

Open your project’s *Startup* class. At the top, there should be the following *using* statement:

```csharp
using ArgentSea.Pg;
```

Then, in the `Startup` class’ `ConfigureServices` method, add:

```csharp
services.AddPgServices(Configuration);
```

This step creates an injectable *SqlServices* object that we can consume in all of our data access clients.

***

## 8. Create a Model Class

A model class has properties that correspond the the fields of a data entity. ArgentSea can automatically map these properties to input or output parameters, the columns of a DataReader object, or (in SQL Server) a table-valued parameter.

For example, suppose your subscriber data can be represented by a class like this:

```csharp
using System;

public class Subscriber
{
    public int SubscriberId { get; set; }

    public string Name { get; set; }

    public DateTime Expiration { get; set; }
}
```

We can simply add mapping attributes to this class:

# [SQL Server](#tab/tabid-sql)

```csharp
using System;
using ArgentSea.Sql;

public class Subscriber
{
    [MapToSqlInt("@SubID")]
    public int SubscriberId { get; set; }

    [MapToSqlNVarChar("@SubscriberName", 255)]
    public string Name { get; set; }

    [MapToSqlDateTime2("@EndDate")]
    public DateTime Expiration { get; set; }
}
```

The “@” parameter prefix is optional — ArgentSea will add the “@” automatically for parameters and remove it automatically when reading data reader columns.

# [PostgreSQL](#tab/tabid-pg)

```csharp
using System;
using ArgentSea.Pg;

public class Subscriber
{
    [MapToPgInteger("subid")]
    public int SubscriberId { get; set; }

    [MapToPgVarchar("subname", 255, true)]
    public string Name { get; set; }

    [MapToPgTimestamp("enddate")]
    public DateTime? Expiration { get; set; }
}
```

***

Note that the property name *does not* need to match the parameter or column name. It is not uncommon for database naming conventions to differ from .NET property naming conventions.

> [!WARNING]
> ArgentSea assumes consistent naming in your data parameters and results. A project with “consistently inconsistent” parameters or column names will find the ArgentSea Mapper of little practical use.

## 5. Define the SQL implementation

# [SQL Server](#tab/tabid-sql)

In the sample [QuickStart](https://github.com/argentsea/quickstarts/blob/master/QuickStart1.Sql/QuickStart1.Sql/Sql/SetupDB.sql), the stored procedure looks like this:

```SQL
CREATE PROCEDURE ws.GetSubscriber
  @SubId int,
  @SubName nvarchar(255) OUTPUT,
  @EndDate datetime2 OUTPUT
AS
BEGIN
  SELECT @SubName = Subscribers.SubName, 
    @EndDate = Subscribers.EndDate
  FROM dbo.Subscribers
  WHERE Subscribers.SubId = @SubId;
END;
```

ArgentSea deliberately tries minimize mixing SQL and .NET code; ideally, only stored procedure names are compiled into the project. Further, to help track which procedures are in use, ArgentSea recommends centralizing the procedure names in a single static `Queries` class:

```csharp
public static class Queries
{
    public static QueryProcedure GetSubscriber => new QueryProcedure("ws.GetSubscriber", new[] { "@SubId", "SubName", "EndDate" });
}
```

Finally, the application needs a repository class to actually retrieve the data. In the sample application, this is called the *SubscriberStore*. This is the class that will call the database stored procedure or SQL statement and return the specified subscriber. If you are creating your own project, you need to construct something similar.

Our implementation of the data access code requires only a few lines:

```csharp
public class SubscriberStore
{
    private readonly SqlDatabases _dbs;
    private readonly ILogger<SubscriberStore> _logger;
    public SubscriberStore(SqlDatabases dbs, ILogger<SubscriberStore> logger)
    {
        _dbs = dbs;
        _logger = logger;
    }

    public async Task<Subscriber> GetSubscriber(int subscriberId, CancellationToken cancellation)
    {
        var db = _dbs["MyDatabase"];
        var prms = new ParameterCollection()
            .AddSqlIntInputParameter("@SubId", subscriberId)
            .CreateOutputParameters<Subscriber>(_logger);
        return await db.Read.MapOutputAsync<Subscriber>(Queries.GetSubscriber, prms, cancellation);
    }
}
```

Because the “@SubId” parameter was manually created, the when the Mapper is enlisted to automatically create all the output parameters, it knows not to duplicate the “@SubId” parameter.

The `MapOutputAsync` method retrieves the data and creates the model instance automatically.

# [PostgreSQL](#tab/tabid-pg)

Create a new folder call “SQL” in your project. In that folder, create a new file named “GetSubscriber.psql”. In Visual Studio, open the file properties and ensure the *Build Action* attribute is set to “None” and the *Copy to Output Directory* attribute is set to “Copy if newer”.

In the simple [QuickStart query](https://github.com/argentsea/quickstarts/blob/master/QuickStart1.Pg/QuickStart1.Pg/Sql/GetCustomer.psql), the statement looks like this:

```SQL
SELECT subscribers.subid, subscribers.subname, subscribers.enddate 
FROM qs1.subscribers 
WHERE subscribers.subid = @subid;
```

ArgentSea deliberately tries minimize mixing SQL and .NET code by storing SQL in files loaded at runtime. To help track which SQL statements are in use, ArgentSea recommends centralizing the procedure names in a single static `Queries` class:

```csharp
public static class Queries
{
    public static QueryStatement GetSubscriber => _getSubscriber.Value;
    private static readonly Lazy<QueryStatement> _getSubscriber = QueryStatement.Create("GetSubscriber", new[] { "subid" });
}
```

Because want to load the file only when it’s needed and we need cache the file without multiple thread all doing the same thing, our declaration of each SQL statement requires two lines of code.

Finally, the application needs a repository class to actually retrieve the data. In the sample application, this is called the *SubscriberStore*. This is the class that will invoke the query and return the specified subscriber. If you are creating your own project, you need to construct something similar.

Our implementation of the data access code requires only a few lines:

```csharp
public class SubscriberStore
{
    private readonly PgDatabases.Database _db;
    private readonly ILogger<SubscriberStore> _logger;

    public SubscriberStore(PgDatabases dbs, ILogger<SubscriberStore> logger)
    {
        _db = dbs["MyDatabase"];
        _logger = logger;
    }

    public async Task<Subscriber> GetSubscriber(int subscriberId, CancellationToken cancellation)
    {
        var prms = new ParameterCollection()
            .AddPgIntegerInputParameter("subid", subscriberId);
        return await _db.Read.MapReaderAsync<Subscriber>(Queries.GetSubscriber, prms, cancellation);
    }
}
```

Because the “subid” parameter was manually created, the when the Mapper is enlisted to automatically create all the output parameters, it knows not to duplicate the “subid” parameter.

The `MapReaderAsync` method retrieves the data and creates the model instance automatically.

***

The ArgentSea database service is injected into the *SubscriberStore* class by the .NET framework. The *SubscriberStore* itself is in turn injected into the controller. Don’t forget to add the *SubscriberStore* class to `services.Configure()` in Startup, so that injection works.

```csharp
public void ConfigureServices(IServiceCollection services)
{
    ...
    services.AddSingleton<SubscriberStore>();
    ...
}
```

The controller for a web API example, can be very simple:

```csharp
[Route("api/[controller]")]
[ApiController]
public class SubscriberController : ControllerBase
{
    private readonly SubscriberStore _store;
    private readonly ILogger<SubscriberController> _logger;

    public SubscriberController(SubscriberStore store, ILogger<SubscriberController> logger)
    {
        _store = store;
        _logger = logger;
    }

    // GET api/subscriber/5
    [HttpGet("{id}")]
    public async Task<ActionResult<Subscriber>> Get(int id, CancellationToken cancellation)
    {
        var result = await _store.GetSubscriber(id, cancellation);
        if (result is null)
        {
            return NotFound();
        }
        return result;
    }
}
```

Due to the work of the Mapper, the controller code would not increase in complexity even if the model had a *many* more properties, each mapped to a parameter or result column.

You should be able to build and run your project. You can test the web service by specifying http://&lt;projectUrl&gr;/api/subscriber/1.

The QuickStart code has a test project that validates that the web service returns the expected results.