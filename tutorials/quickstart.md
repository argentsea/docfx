# Quickstart

This article will breeze you through a simple setup of ArgentSea for non-sharded data access. 
It assumes that you are using *.NET Core* in *Visual Studio*.

If you get stuck or have questions, click on one of the 
links to the other more in-depth articles.

## 1. Create a Project (or use an existing one)
A sample quickstart project is [here](https://github.com/argentsea/quickstarts). 
It is based on the ASP.NET API project type.

If you create an empty project, you will need to ensure that appsettings.json is added. 

## 2. Add ArgentSea to your project
Use NuGet to add ArgentSea to your project. 
Select the package that corresponds to your database platform.

As of today, you can add one of the following NuGet packages:
* For SQL Server databases, use **ArgentSea.Sql**
* For PostgreSQL, use **ArgentSea.Pg**
Both packages will automatically include the shared ArgentSea package 
and any other dependencies. 

You can learn more about adding a reference to ArgentSea [here](https://docs.microsoft.com/en-us/nuget/quickstart/install-and-use-a-package-in-visual-studio).

## 3. Define your Login Information
There are two required configuration sections. The first of these provides security information.

If you have [UserSecrets](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets) 
set up (preferred), add the json below to your User Secrets file (right-click
on the project and select *Manage User Secrets*).

If you are not using User Secrets, you can simply one of the json sections to your appsettings.json 
configuration file.

> [!WARNING]
> The sample application *does* use User Secrets, so if you are following along at home you 
> will need to manually copy the credentials section below into the secrets.json file in the sample app.

For a connection using username and password authentication, use:
```
"Credentials": [
  {
    "SecurityKey": "SecKey1",
    "UserName": "webuser",
    "Password": "Pwd123456"
  }
]
```
Change the values, as appropriate, to represent a valid login.

Alternatively, for a connection using Windows authentication, use:
```
  "Credentials": [
    {
      "SecurityKey": "SecKey1",
      "WindowsAuth": true
    }
  ],
```

> [!NOTE]
> Storing passwords more securely is an incidental benefit of having a separate
> Credentials section. The principal purpose is to simplify login management 
> when the system has many connections across multiple shard sets.

> [!WARNING]
> In a production deployment, the Credential section could be hosted in Azure Key Vault, 
> AWS Secrets Manager, or some other secure resource.

## 4. Define Your Database Settings
The database settings tell ArgentSea how to build the (non-security) part of your connection 
string. Any attribute not specified will use a default value, so the required information 
is quite minimal.

In your appsettings.json file, add the following section:

# [SQL Server](#tab/tabid-sql)
Configure the *DataSource* and *InitialCatalog* properties below.
```
  "SqlDbConnections": [
    {
      "DatabaseKey": "MyDatabase",
      "SecurityKey": "SecKey1",
      "DataConnection": {
        "DataSource": "localhost",
        "InitialCatalog": "MyDb"
      }
    }
  ]
```
# [PostgreSQL](#tab/tabid-pg)
Configure the *Host* and *Database* properties below.
```
  "PgDbConnections": [
    {
      "DatabaseKey": "MyDatabase",
      "SecurityKey": "SecKey1",
      "DataConnection": {
        "Host": "localhost",
        "Database": "MyDb"
      }
    }
  ]
```
***
A deep-dive into configuration settings, including a complete list of configuration attributes,
is available [here](/tutorials/configuration.html).

> [!NOTE]
> In a production deployment, environment-specific connection information might be stored in 
> server environment variables rather than configuration files. This make deployments
> much easier to manage.

## 6. Load ArgentSea on Application Start
Open your project’s *Startup* class. At the top, add the following *using* statment:
# [SQL Server](#tab/tabid-sql)
```
using ArgentSea.Sql;
```
Then, in the `Startup` class’ `ConfigureServices` method, add:
```
services.AddSqlServices(Configuration);
```
This step creates an injectable *SqlServices* object that we can use in all
of our data access clients.
# [PostgreSQL](#tab/tabid-pg)
```
using ArgentSea.Pg;
```
Then, in the `Startup` class’ `ConfigureServices` method, add:
```
services.AddPgServices(Configuration);
```
This step creates an injectable *PgServices* object that we can use in all
of our data access clients.
***


## 7. Create a Model Class
A model class has properties that correspond the the fields of a data entity. 
ArgentSea can automatically map these properties to input or output parameters,
the columns of a datareader object, or (in SQL Server) a table-valued parameter.

For example, suppose your subscriber data can be represented by a class like this:
```
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
```
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
The “@” parameter prefix is optional — ArgentSea will add the “@” automatically for parameters
and remove it automatically when reading data reader rows.
# [PostgreSQL](#tab/tabid-pg)

```
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
> [!NOTE]
> Note that the property name *does not* need to match the parameter or column name.
> It is not uncommon for database naming conventions to differ from .NET property 
> naming conventions.

> [!WARNING]
> ArgetSea assumes consistant naming in your data parameters and results. 
> A project with consistently inconsistent parameters or column names will find 
> the ArgentSea Mapper of little help.

## 5. Call a Stored Procedure or Function
Create one more class, called *SubscriberStore*. This is the class that will call the database
stored procedure or function and return the specified subscriber.

# [SQL Server](#tab/tabid-sql)
Our very simple stored procedure can be something like this:
```
CREATE PROCEDURE dbo.GetSubscriber (@SubscriberId int)
As
BEGIN
    SELECT Subscribers.SubscriberId, Subscribers.SubscriberName, Subscribers.EndDate
    FROM dbo.Subscribers
    WHERE Subscribers.SubscriberId = @SubscriberId;
END;
```
You can find the complete project setup SQL [here](https://github.com/argentsea/quickstarts/blob/master/QuickStart1.Sql/QuickStart1.Sql/Sql/SetupDB.sql).

Our implementation of the data access code requires only a few lines:
```
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
    var db = _dbs.DbConnections["MyDatabase"];
    var prms = new QueryParameterCollection()
      .AddSqlIntInParameter("@SubId", subscriberId);
    Mapper.MapToOutParameters(prms, typeof(Subscriber), _logger);
    return await db.QueryAsync<Subscriber>("ws.GetSubscriber", prms, cancellation);
  }
}
```
# [PostgreSQL](#tab/tabid-pg)
```
Our very simple PostgreSQL function can be something like this:

CREATE FUNCTION ws.GetSubscriber (
  _subid integer,
  OUT _subname varchar(255),
  OUT _enddate timestamp 
)
SECURITY DEFINER
AS $$
  SELECT Subscribers.SubName,
    Subscribers.EndDate
  FROM Subscribers
  WHERE Subscribers.SubId = _subid;
$$ LANGUAGE sql;
```
You can find the complete project setup SQL [here](https://github.com/argentsea/quickstarts/blob/master/QuickStart1.Pg/QuickStart1.Pg/Sql/SetupDB.sql).

Our implementation of the data access code requires only a few lines:
```public class SubscriberStore
{
  private readonly PgDatabases _dbs;
  private readonly ILogger<SubscriberStore> _logger;
  public SubscriberStore(PgDatabases dbs, ILogger<SubscriberStore> logger)
  {
    _dbs = dbs;
    _logger = logger;
  }

public async Task<Subscriber> GetSubscriber(int subscriberId, CancellationToken cancellation)
{
  var db = _dbs.DbConnections["MyDatabase"];
  var prms = new QueryParameterCollection()
    .AddPgIntegerInParameter("_subid", subscriberId);
  Mapper.MapToOutParameters(prms, typeof(Subscriber), _logger);
  return await db.QueryAsync<Subscriber>("ws.GetSubscriber", prms, cancellation);
}
```
***
The ArgentSea database service is injected into the *SubscriberStore* class. 
Don’t forget to add the *SubscriberStore* class to `services.Configure()` in Startup, 
so that it can be injected to any classes that need to access a subscriber.

The code only needs to set the required input parmemeter value (the subscriber id to be retrieved). 
The Mapper sets the remaining parameters automatically. 
The QueryAsync method retrieves the data and creates the model instance automatically.
 
The controller for a web API example, can be very simple:
```
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

  [HttpGet("{id}")]
  public async Task<ActionResult<Subscriber>> Get(int id, CancellationToken cancellation)
  {
    return await _store.GetSubscriber(id, cancellation);
  }
}

```
> [!NOTE]
> Due to the work of the Mapper, the controller code would not increase in complexity
> even if the model had a hundred properties mapped to a hundred parameters.
