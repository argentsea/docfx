# QuickStart One

This article will step you through a simple setup of ArgentSea for non-sharded data access. This presentation introduces concepts which are further elaborated in the [subsequent article](quickstart2.md). It assumes that you are using *.NET Core* in some flavor of *Visual Studio*.

If you get stuck or have questions, click on one of the links to the other, more in-depth, articles.

## 1. Create a Project (or use an existing one)

A sample QuickStart project is [here](https://github.com/argentsea/quickstarts). It is based on the ASP.NET API project type.

If you prefer to start by creating a new, empty project, ensure that appsettings.json is added.

## 2. Setup your Database

If you are using the QuickStart1 project, a setup script is including in the project. You can run the script against the database server to create the sample database.

Since ArgentSea works with stored procedures, it can use an account that has only EXECUTE permissions. An elegant way to implement this is to create a distinct schema, then grant the user EXECUTE permission only to that schema. Henceforth, any procedure or function belonging to that schema can be executed by this client — but nothing else.

> [!CAUTION]
> A password is embedded in the setup script (and corresponds to the credential sections below); you might consider changing this value. However, this is also a low-privileged user, so it is not high-risk if you leave things as they are.

## 3. Add ArgentSea to your project

If you have loaded the QuickStart project, you can skip this step. If you are creating a new project, use NuGet to add ArgentSea to your project. Select the package that corresponds to your database platform.

As of today, you can add one of the following NuGet packages:

* For Microsoft SQL Server databases, use **ArgentSea.Sql**
* For PostgreSQL, use **ArgentSea.Pg**

Both packages will automatically include the shared ArgentSea package and any other dependencies. Using both packages in the same project is not a tested scenario.

You can learn more about adding a reference to ArgentSea [here](https://docs.microsoft.com/en-us/nuget/quickstart/install-and-use-a-package-in-visual-studio).

## 4. Define your Login Information

ArgentSea leverages the .NET Core configuration architecture, which means that the configuration information is combined from multiple providers. In this example, we will store most connection information in the *appsettings.json* file, but store the login password securely in a separate store.

> [!TIP]
> Because the new configuration architecture in .NET core allows values to be hosted in multiple places, we can also use environment variables — which can be very useful for managing a release pipeline. In that case, we might store the server or database information there, instead of *appsettings.json*. Again, the details are described in the [configuration](/tutorials/configuration.md) tutorial.

If you are creating a new project, add [UserSecrets](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets) to your project. This provides a means of keeping passwords out of your source control. You can add UserSecrets to your project by loading the *Microsoft.Extensions.Configuration.UserSecrets* NuGet package.

> [!TIP]
> Because the QuickStart1 sample application uses User Secrets, if you are following along at home with the downloaded project, you will still need to manually copy the credentials to the User Secrets in the sample app.

If you are using UserSecrets, right-click on the project and select *Manage User Secrets* to open the secrets JSON file. Otherwise, you can simply add the security information to your appsettings.json file.

# [SQL Server](#tab/tabid-sql)

To connect using username and password authentication, add this to your User Secrets:

```json
{
  "SqlDbConnections": [
    {
      "Password": "<password>"
    }
  ]
}
```

Change the values, as appropriate, to represent a valid login.

Configure the *DataSource* and *InitialCatalog* properties in your *appsettings.json* configuration file:

```json
  "SqlDbConnections": [
    {
      "DatabaseKey": "MyDatabase",
      "DataSource": "localhost",
      "InitialCatalog": "MyDb"
    }
  ]
```

Note that if you are using Windows authentication, you can just specify this in *appsettings.json* and you don’t need to manage secrets:

```json
  "SqlDbConnections": [
    {
      "DatabaseKey": "MyDatabase",
      "DataSource": "localhost",
      "InitialCatalog": "MyDb",
      "WindowsAuth": true
    }
  ]
```

# [PostgreSQL](#tab/tabid-pg)

To connect using username and password authentication, add this to your User Secrets:

```json
{
  "SqlDbConnections": [
    {
      "Password": "<password>"
    }
  ]
}
```

Configure the *Host* and *Database* properties in your *appsettings.json* configuration file:

```json
  "PgDbConnections": [
    {
      "DatabaseKey": "MyDatabase",
      "Host": "localhost",
      "Database": "MyDb"
    }
  ]
```

Note that if you are using Windows authentication, you can just specify this in *appsettings.json* and you don’t need to manage secrets:

```json
  "SqlDbConnections": [
    {
      "DatabaseKey": "MyDatabase",
      "Host": "localhost",
      "Database": "MyDb",
      "WindowsAuth": true
    }
  ]
```

***


Alternatively, for a connection using Windows authentication, use:

```json
  "Credentials": [
    {
      "SecurityKey": "SecKey1",
      "WindowsAuth": true
    }
  ],
```

***

> [!CAUTION]
> In a production deployment, the Credential section should be hosted in [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/), [Azure Key Vault](https://azure.microsoft.com/en-us/services/key-vault/), or some other secure resource.

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

The “@” parameter prefix is optional — ArgentSea will add the “@” automatically for parameters and remove it automatically when reading data reader rows.

# [PostgreSQL](#tab/tabid-pg)

```csharp
using System;
using ArgentSea.Pg;

public class Subscriber
{
    [MapToPgInteger("SubId")]
    public int SubscriberId { get; set; }

    [MapToPgVarChar("SubscriberName", 255)]
    public string Name { get; set; }

    [MapToPgTimestamp("EndDate")]
    public DateTime Expiration { get; set; }
}
```

***

Note that the property name *does not* need to match the parameter or column name. It is not uncommon for database naming conventions to differ from .NET property naming conventions.

> [!WARNING]
> ArgentSea assumes consistent naming in your data parameters and results. A project with “consistently inconsistent” parameters or column names will find the ArgentSea Mapper of little practical use.

## 5. Call a Stored Procedure or Function

Your application needs a repository class to actually retrieve the data. In the sample application, this is called the *SubscriberStore*. This is the class that will call the database stored procedure or function and return the specified subscriber. If you are creating your own project, you need to construct something similar.

# [SQL Server](#tab/tabid-sql)

In the simple [QuickStart](https://github.com/argentsea/quickstarts/blob/master/QuickStart1.Sql/QuickStart1.Sql/Sql/SetupDB.sql), the stored procedure looks like this:

```SQL
CREATE PROCEDURE dbo.GetSubscriber (@SubscriberId int)
As
BEGIN
    SELECT Subscribers.SubscriberId, Subscribers.SubscriberName, Subscribers.EndDate
    FROM dbo.Subscribers
    WHERE Subscribers.SubscriberId = @SubscriberId;
END;
```

Our implementation of the data access code requires only a few lines:

```csharp
public class SubscriberStore
{
    private readonly SqlDatabases.DataConnection _db;
    private readonly ILogger<SubscriberStore> _logger;
    public SubscriberStore(SqlDatabases dbs, ILogger<SubscriberStore> logger)
    {
        _db = dbs["MyDatabase"];
        _logger = logger;
    }

    public async Task<Subscriber> GetSubscriber(int subscriberId, CancellationToken cancellation)
    {
        var parameters = new QueryParameterCollection()
            .AddSqlIntegerInputParameter("@subid", subscriberId)
            .CreateOutputParameters<Subscriber>(_logger);
        return await _db.MapOutputAsync<Subscriber>("ws.GetSubscriber", parameters, cancellation);
    }
}
```

# [PostgreSQL](#tab/tabid-pg)

In the simple [QuickStart](https://github.com/argentsea/quickstarts/blob/master/QuickStart1.Pg/QuickStart1.Pg/Sql/SetupDB.sql), the function looks like this:

```SQL
CREATE OR REPLACE FUNCTION ws.GetSubscriber (
  _subid integer,
  OUT _subname varchar(255),
  OUT _enddate timestamp 
)
SECURITY DEFINER
AS $$
  SELECT Subscribers.SubName,
    Subscribers.EndDate
  FROM qs1.Subscribers
  WHERE Subscribers.SubId = _subid;
$$ LANGUAGE sql;
```

Our implementation of the data access code requires only a few lines:

```csharp
public class SubscriberStore
{
    private readonly PgDatabases.DataConnection _db;
    private readonly ILogger<SubscriberStore> _logger;
    public SubscriberStore(PgDatabases dbs, ILogger<SubscriberStore> logger)
    {
        _db = dbs["MyDatabase"];
        _logger = logger;
    }

    public async Task<Subscriber> GetSubscriber(int subscriberId, CancellationToken cancellation)
    {
        var prms = new QueryParameterCollection()
            .AddPgIntegerInputParameter("_subid", subscriberId)
            .CreateOutputParameters<Subscriber>(_logger);
        return await _db.MapOutputAsync<Subscriber>("ws.GetSubscriber", prms, cancellation);
    }
}
```

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

The code only needs to set the required input parameter value (the subscriber id to be retrieved). The Mapper sets the remaining parameters automatically. The QueryAsync method retrieves the data and creates the model instance automatically.

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

Due to the work of the Mapper, the controller code would not increase in complexity even if the model had a hundred properties mapped to a hundred parameters.

You should be able to build and run your project. You can test the web service by specifying http://<projectUrl>/api/subscriber/1.

The QuickStart code has a test project that validates that the web service returns the expected results.