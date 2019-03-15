# QuickStart Two

The [previous QuickStart](configuration.md) introduced configuration and mapping. This tutorial extends that information while working with a sharded data set. This tutorial also extends the mapping functionality to include list and object properties on the model class.

Sharded data introduces two complexities:

* How do I uniquely identify and locate a record, which could be on any shard?
* How do I manage data on one shard and related data on a foreign shard?

This walkthrough illustrates how both challenges can be met.

## Create the Project

If you are following along at home with a new project, in Visual Studio create a new “ASP.NET Core Web Application” project. When prompted, select the “API” project type. Once the solution is created, open your dependencies and add the following NuGet packages:

* __ArgentSea.Sql__ or __ArgentSea.Pg__ - for SQL Sever or PostgreSQL databases respectively
* __Swashbuckle.Aspnetcore__ - for [Swagger](https://swagger.io/) and for invoking the API without creating a client

To follow a standard convention for an MVC project, create folders for __Models__, __InputModels__, and __Stores__ (or “Repositories” if you prefer).

## The Sample Data

Whether you simply downloaded the walkthrough or a creating a new project, you will need to create some sample shards.

Our sample application is going to track __Customers__. The data set is completely made-up and not especially realistic. __Customers__ can have multiple __Locations__ (1:∞). __Customers__ can also have __Contacts__, but the __Contacts__ can belong to more than one __Customer__ (∞:∞). The __Contact__ may not exist in the same shard as the __Customer__.

> [!NOTE]
> The data is set up this way to illustrate one of the difficulties with sharded data: managing relationships between records that exist on different shards. In this case, a __Customer__ may be associated with a Contact on any shard. Managing this — an managing this efficiently — adds substantial complexity.

In our sample model, the ∞:∞ relationship between a __Contact__ and a __Customer__ is managed by a linking table, __CustomerContacts__. Given the id of a local __Customer__, the table lists the keys of the associated __Contacts__. What happens if instead I have the id of a __Contact__ and want to find all of the __Customers__? With only that table, it would be necessary to query *every shard* to determine whether they have a related __Customer__!

To better optimize this, the data model creates a second __ContactCustomers__ linking table. With this approach, if I have a __Contact__ and want to find __Customers__ (including those in foreign shards), or __Customers__ and want to find __Contacts__ (again, even those not local), I can use the appropriate linking table and query only the shards with relevant records data. The downside is that when the __Customer__ and __Contact__ are on different shards, this relationship must be managed in different tables on different databases.

In a real implementation, it might be beneficial to include additional information in the linking tables, like including the contact name within the __CustomerContact__ table for example, so that common lookups do not require secondary lookups from related shards just to get the contact name. The QuickStart example doesn’t do this so that it can illustrate performing these secondary lookups.

Naturally, deleting a __Customer__ means removing the __ContactCustomers__ link from each Contact also, which could be on any shard. This shard list must be retrieved from the __CustomerContacts__ list before it is deleted. Updating a Customer’s __Contacts__ could also impact multiple shards in two ways: a removed foreign-shard __Contact__ must be deleted and a additional foreign-shard __Contacts__ must be added.

## [SQL Server](#tab/tabid-sql)

The SQL for the sample data is found in the GitHub source repository, at https://github.com/argentsea/quickstarts/tree/master/QuickStart2.Sql/QuickStart2.Sql/SqlSetup.

The first SQL script to execute is *ServerSetup.sql*, which creates four databases and two logins.

> [!CAUTION]
> The logins contain weak passwords (that are published on the Internet), so you might consider changing them; on the other hand, these will only have permission to execute procedures or functions in a specific namespace, so it’s not a big risk.

> [!NOTE]
> Conceptually, these four databases would correspond to regional data stores in the *United States*, *Brazil*, *Europe*, and *China*. In my real global application I would replicate the data from each region to each other region. Therefore, each region would have one writable data store and three readable ones. In this way, the “local” shard is writeable, the “foreign” shards are read-only. Writes to a foreign shard must be done across the WAN. Your implementation may vary.  
> Our walkthrough, however, only needs four simple databases on one server. We’ll only imagine the rest.

Next, after the databases have been created, *connect to each database in turn* and run the *ShardSetup.sql* SQL script. This will created the schemas, tables, stored procedures, reference data, etc. for our sample databases.

Finally, run the shard-specific SQL scripts — *ShardUS.sql*, *ShardBR.sql*, *ShardUS.sql*, *ShardUS.sql* — within their respective databases. This will load the shard-specific sample data.

## [PostgreSQL](#tab/tabid-pg)

The SQL for the sample data is found in the GitHub source repository, at https://github.com/argentsea/quickstarts/tree/master/QuickStart2.Pg/QuickStart2.Pg/SqlSetup.

Create four PostgreSQL databases:

* customershard_br
* customershard_eu
* customershard_us
* customershard_zh

> [!NOTE]
> Conceptually, these four databases would correspond to regional data stores in the *United States*, *Brazil*, *Europe*, and *China*. In my real global application I would replicate the data from each region to each other region. Therefore, each region would have one writable data store and three readable ones. In this way, the “local” shard is writeable, the “foreign” shards are read-only. Writes to a foreign shard must be done across the WAN. Your implementation may vary.  
> Our walkthrough, however, only needs four simple databases on one server. We’ll only imagine the rest.

After the databases have been created, *connect to each database in turn* and run the *ShardSetup.sql* SQL script. This will create the schemas, tables, stored procedures, reference data, etc. for our sample databases. 

> [!CAUTION]
> The database setup create users with weak passwords (that are also published on the Internet), so you might consider changing them.

Finally, run the shard-specific SQL scripts — *ShardUS.sql*, *ShardBR.sql*, *ShardUS.sql*, *ShardUS.sql* — within their respective databases. This will load the shard-specific sample data.

***

At this point, we should have four database with identical table structures. SQL Server instances should also have a identical set of stored procedures.

## The ShardId

Each of the four databases needs an identifier. This is more significant than it sounds. ArgentSea uses a “virtual compound key” consisting of the ShardId and the RecordId; this is called a [ShardKey](/api/ArgentSea.ShardKey-2.html). A thorough discussion of the options and impact is [here](/tutorials/sharding/shardkey.md). The précis is that once established, the type (and values) of the *ShardId* cannot be easily changed. The ShardId is a part of record identification, so confusion or misconfiguration could result in data corruption.

## [SQL Server](#tab/tabid-sql)

In this sample application, the type of the ShardId for SQL Server is a `byte` (SQL TinyInt). Because ArgentSea uses a generic type to define the ShardId, you can use one of many types in your application; however, the ShardId must be consistent throughout the application.

Naturally, this mandates an upper maximum  of 256 shards. If there is any possibility of one day needing to exceeding this count, you can use a `short` data type instead.

## [PostgreSQL](#tab/tabid-pg)

In this sample application, the type of the ShardId for PostgreSQL is a `short` (SQL SmallInt). Because ArgentSea uses a generic type to define the ShardId, you can use one of many types in your application; however, the ShardId must be consistent throughout the application.

***

## Configuring Connections

Because sharded data may require a large number of data connections, ArgentSea offers a more flexible way of managing this than by using connection strings. ArgentSea offers the “Hereditary Configuration Hierarchy”. This allows you to set an attribute at the parent level and all children will inherit this value, unless overwritten by a child. A more thorough discussion is [here](/tutorials/configuration/hierarchy.md).

In our sample application, we can use the same server or host for all connections; each shard connection only changes the database name. (In a production deployment, the configuration might be exactly backwards: the databases have identical names, but each is on a different host). We also want to use the *webWriter* user for write connections and *webReader* for read connections.

So the configuration settings looks like this (with annotations):

## [SQL Server](#tab/tabid-sql)

```json
{
  "SqlShardSets": [¹
    {
      "ShardSetName": "Customers",²
      "DataSource": ".",³
      "DefaultShardId": 1,⁴
      "Write": {⁵
        "UserName": "webWriter",
        "Password": "Pwd567890",
      },
      "Read": {⁵
        "ApplicationIntent": "ReadOnly",
        "UserName": "webReader",
        "Password": "Pwd123456"
      },
      "Shards": [⁶
        {
          "ShardId": 1,⁷
          "InitialCatalog": "CustomerShardUS"⁸
        },
        {
          "ShardId": 2,⁷
          "InitialCatalog": "CustomerShardEU"⁸
        },
        {
          "ShardId": 3,⁷
          "InitialCatalog": "CustomerShardBR"⁸
        },
        {
          "ShardId": 4,⁷
          "InitialCatalog": "CustomerShardZH"⁸
        }
      ]
    }
  ]
}
```

### Annotations

¹ `SqlShardSets` is the root JSON section for all the shard configuration metadata. It contains an array of shard sets.

² `ShardSetName` is a *required* key for this specific shard set. Multiple shard sets are possible and each will be identified by this key. This value must exactly match the value used in your code to invoke this shard set.

³ `DataSourceName` is a connection attribute. Connection attributes can appear anywhere in the hierarchy. Because it appears at the “shard set” level, all shards in the shard set will inherit this server name.

⁴ `DefaultShardId` this setting determins which shard is presented as the `ShardSet.DefaultShard`. This is useful for determining which shard should be actively accepting new records for this client.

⁵ `Read` and `Write` are peculiar, and optional, members of shard set’s “inheritance” chain, as their children are indirect. Any attributes defined in the shard set’s `Write` section apply only to write connections. Likewise, for `Read` connections. These values can be overwritten by shard or connection attributes.

⁶ `Shards` is an array of shard connections, one for each shard in the shard set.

⁷ `ShardId` is a *required* identifier for the shard. This value is essential for finding and identifying a sharded record. It cannot be duplicated within a shard set. If the type of the ShardId is a string, then this value should have quotes around it in the JSON file.

⁸ `InitialCatalog` is a connection attribute. Because it appears at the shard level, both read connections and write connections for this shard will inherit this value.

## [PostgreSQL](#tab/tabid-pg)

```json
{
  "PgShardSets": [¹
    {
      "ShardSetName": "Customers",²
      "DefaultShardId": 1,⁴
      "Host": "localhost",³
      "Write": {⁵
        "UserName": "webWriter",
        "Password": "Pwd567890"
      },
      "Read": {⁵
        "UserName": "webReader",
        "Password": "Pwd123456"
      },
      "Shards": [⁶
        {
          "ShardId": 1,⁷
          "Database": "CustomerShardUS"⁸
        },
        {
          "ShardId": 2,⁷
          "Database": "CustomerShardEU"⁸
        },
        {
          "ShardId": 3,⁷
          "Database": "CustomerShardBR"⁸
        },
        {
          "ShardId": 4,⁷
          "Database": "CustomerShardZH"⁸
        }
      ]
    }
  ]
}
```

### Annotations

¹ `PgShardSets` is the root JSON section for all the shard configuration metadata. It contains an array of shard sets.

² `ShardSetName` is a *required* key for this specific shard set. Multiple shard sets are possible and each will be identified by this key. This value must exactly match the value used in your code to invoke this shard set.

³ `Host` is a connection attribute. Connection attributes can appear anywhere in the hierarchy. Because it appears at the “shard set” level, all shards in the shard set will inherit this server name.

⁴ `DefaultShardId` this setting determins which shard is presented as the `ShardSet.DefaultShard`. This is useful for determining which shard should be actively accepting new records for this client.

⁵ `Read` and `Write` are peculiar, and optional, members of shard set’s “inheritance” chain, as their children are indirect. Any attributes defined in the shard set’s `Write` section apply only to write connections. Likewise, for `Read` connections. These values can be overwritten by shard or connection attributes.

⁶ `Shards` is an array of shard connections, one for each shard in the shard set.

⁷ `ShardId` is a *required* identifier for the shard. This value is essential for finding and identifying a sharded record. It cannot be duplicated within a shard set. If the type of the ShardId is a string, then this value should have quotes around it in the JSON file.

⁸ `Database` is a connection attribute. Because it appears at the shard level, both read connections and write connections for this shard will inherit this value.

***

This hierarchy, then, defines a server name once, to be used for the entire shard set. The read and write logins are also defined once, to be used by all read or write connections in the shard set. Each shard has a distinct database name. ArgentSea can build read and write connections to each data store without the need to configure any of this data redundantly — the login, server name, and database names are each managed only once.

When you save this configuration to project’s appsettings file, be sure to update the JSON to the appropriate server references.

You might consider moving the login password information to the UserSecrets store, which is a best practice. Simply remove the password entries from the appsettings.json hierarchy and add them to the usersecrets.json file. Ideally, the password should also be changed to a different value.

## [SQL Server](#tab/tabid-sql)

### User Secrets Entry

```json
{
  "SqlShardSets": [
    {
      "Write": {
        "Password": "Pwd567890",
      },
      "Read": {
        "Password": "Pwd123456"
      }
    }
  ]
}
```

## [PostgreSQL](#tab/tabid-pg)

### User Secrets Entry

```json
{
  "PgShardSets": [
    {
      "Write": {
        "Password": "Pwd567890",
      },
      "Read": {
        "Password": "Pwd123456"
      }
    }
  ]
}
```

***

>[!WARNING]
> The configuration arrays in appsettings.json and usersecrets.json will not match if they do not appear in exactly the same order. In this sample, we have only one shard set and the passwords are not in the `Shards` array, so this is not a concern.

## Creating the Models

The process of creating a model class was introduced in [Quickstart 1](configuration.md). Essentially, it simply requires adding attributes to properties, which the Mapper can then use. This QuickStart adds four new wrinkles: shard keys, object properties, list properties, and inheritance.

The complete code is on GitHub, at https://github.com/argentsea/quickstarts/tree/master/QuickStart2.Sql/QuickStart2.Sql for SQL Server, or https://github.com/argentsea/quickstarts/tree/master/QuickStart2.Pg/QuickStart2.Pg for PostgreSQL. The classes and SQL you need are located there; it is not fully reproduced here.

## [SQL Server](#tab/tabid-sql)

To use the attributes, each Model class should include a `using ArgentSea.Sql` statement.

Optionally, a second `using` statement can reduce the redundancy of declaring generic arguments throughout the Model class.

```csharp
using ArgentSea.Sql;
using ShardKey = ArgentSea.ShardKey<byte, int>;
```

Because the generic ShardId type cannot change within an application, this second `using` statement can simplify the Model code. Of course, this simplification won't work if you use different data types for the record id (integer vs bigint) in different sharded tables in the same model.

You can implement an equivalent practice for the [ShardChild](/api/ArgentSea.ShardChild-3.html) object too. 

(While it might seem even better to declare a subclass that inherits from ShardKey or ShardChild which defines the ShardId type for your entire project, unfortunately, the ShardKey and ShardChild are *structs*, so inheritance is not an option.)

## [PostgreSQL](#tab/tabid-pg)

To use the attributes, each Model class should include a `using ArgentSea.Pg;` statement.

Optionally, a second `using` statement can reduce the redundancy of declaring generic arguments throughout the Model class.

```csharp
using ArgentSea.Pg;
using ShardKey = ArgentSea.ShardKey<short, int>;
```

Because the generic ShardId type cannot change within an application, this second `using` statement can simplify the Model code. Of course, this simplification won't work if you use different data types for the record id (integer vs bigint) in different sharded tables in the same model.

You can implement an equivalent practice for the [ShardChild](/api/ArgentSea.ShardChild-3.html) object too. 

(While it might seem even better to declare a subclass that inherits from ShardKey or ShardChild which defines the ShardId type for your entire project, unfortunately, the ShardKey and ShardChild are *structs*, so inheritance is not an option.)

***

## Advanced Model Mapping

The previous walthrough demonstrated mapping to standard .NET types like strings, numbers, and dates. This walkthrough illustrates mapping to objects, lists, and child classes.

### Properties with Object Types

Our data contains __Location__ data with *latitude* and *longitude* values. Generally, these are usually managed as a value pair. Geographic functions would likely expect a single geographic *coordinates* argument, rather than the two separate values. It would be handy to map data directly to/from a coordinates class, which would be a property of the __Location__ class.

That is exactly what the `MapToModel` attribute does. This attribute tells the mapper that the property is a child object that also has properties to be included in the mapping.

## [SQL Server](#tab/tabid-sql)

```csharp
// The coordinates class:
public class CoordinatesModel
{
    [MapToSqlFloat("Latitude")]
    public double Latitude { get; set; }

    [MapToSqlFloat("Longitude")]
    public double Longitude { get; set; }
}

// The location, which contains the coordinates class as a property:
public class LocationModel
{
    //include other properties here...

    [MapToModel]
    public CoordinatesModel Coordinates { get; set; }
}

```

## [PostgreSQL](#tab/tabid-pg)

```csharp
// The coordinates class:
public class CoordinatesModel
{
    [MapToPgDouble("latitude")]
    public double Latitude { get; set; }

    [MapToPgDouble("longitude")]
    public double Longitude { get; set; }
}

// The location, which contains the coordinates class as a property:
public class LocationModel
{
    //include other properties here...

    [MapToModel]
    public CoordinatesModel Coordinates { get; set; }
}

```

***

If the Coordinates property is null, the Mapper will instantiate an instance before setting the properties. Of course, the CoordinatesModel must have a default constructor and the property must be settable. If you want to make the property read-only, just make sure that the Coordinates object exists:

```csharp
[MapToModel]
public CoordinatesModel Coordinates { get; } = new CoordinatesModel();
```

### Properties with List Types

One of the most expensive activities an application can do is reach out to another server. Our high-performance application should do everything possible to minimize database server round-trips. This means getting all the data necessary to populate our __Customer__ model in a single request. The ArgentSea Mapper can automatically handle multiple results from a single request.

Our __Customer__ can have any number of __Locations__. Our __Customer__ can also have any number of __Contacts__. Our query returns the __Customer__, __Location__ *and* __Contact__ information in a single round-trip.

## [SQL Server](#tab/tabid-sql)

> [!TIP]
> The base __Customer__ record *could* be returned in either output parameters or in a single-row SELECT result. The first would use the Mapper’s `MapOutput` method, the other requires the `MapReader` method; both would use the data reader to handle list properties.

To map the multiple data reader results to the Model, we tell the Mapper the order of the results when we fetch:

```csharp
var prms = new QueryParameterCollection()
    .AddSqlIntInputParameter("@CustomerId", customerKey.RecordId)
    .CreateOutputParameters<CustomerModel>(_logger);
return await _shardSet[customerKey].Read.MapOutputAsync<CustomerModel, LocationModel, ContactModel>(Queries.CustomerGet, prms, cancellation);
```

The `CustomerModel` type in the first generic position tells the Mapper that that is the base object type. The Mapper will automatically create a new instance of the CustomerModel type and populate its properties from the query’s output parameters.

 The `LocationModel` type in the next generic position indicates that the first data reader result contains this type. The Mapper will build a list of locations, find a property of type `List<LocationModel>` or `IList<LocationModel>`, and set the property to the list object.

Likewise, the third generic argument tells the Mapper that the next data reader result is a list of __Contacts__, which the Mapper will use to populate the __Contacts__ property.


## [PostgreSQL](#tab/tabid-pg)

To map the multiple data reader results to the Model, we tell the Mapper the order of the results when we fetch:

```csharp
var prms = new ParameterCollection()
    .AddPgIntegerInputParameter("customerid", customerKey.RecordId);
return await _shardSet[customerKey].Read.MapReaderAsync<CustomerModel, CustomerModel, LocationModel, ContactModel>(Queries.CustomerGet, prms, cancellation);
```

> [!WARNING]
> Although you can also capture database results using output parameters by using `MapOutputAsync`, this is not recommended. Unlike SQL Server, there is no performance benefit with this approach and this will error if multiple SQL statements are used in the query.

The `CustomerModel` type in the first generic position tells the Mapper that that is the base object type. The remaining generic arguments inform the mapper of the order in the query results. Because our customer data is returned in the first result, the `CustomerModel` appears again in the second position of the generic argument list.

The `LocationModel` type in the third generic position indicates that the second data reader result contains this type. The Mapper will build a list of locations, find a property of type `List<LocationModel>` or `IList<LocationModel>`, and set the property to the list object. 

Likewise, the forth generic argument tells the Mapper that the third data reader result is a list of __Contacts__, which the Mapper will used to populate the Contacts property.

***

Only a few lines of code are all that is required to manage this complex result.

### Model Inheritance

The sample application has a `CustomerListItem` Model, which contains a record key and a customer name property. The `CustomerModel` include those same values, plus some others. By inheriting from first Model, the `CustomerModel` inherits the key and an customer name, including their mapping attributes.

Because database queries often return subsets of entity columns, this object inheritance technique is useful in allowing the mapping attributes to be defined only once.

### The ShardKey

The final object type which may combine multiple data records is the [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) types. These are described in detail [here](/tutorials/sharding/shardkey.md).

## [SQL Server](#tab/tabid-sql)

```csharp
[MapShardKey('c', "@CustomerId")]
[MapToSqlInt("@CustomerId")]
public ShardKey CustomerKey { get; set; }
```

## [PostgreSQL](#tab/tabid-pg)

```csharp
[MapShardKey('c', "customerid")]
[MapToPgInteger("customerid")]
public ShardKey CustomerKey { get; set; }
```

***

> [!NOTE]
> The ShardKey in this example does not specify a ShardId data mapping. Because the client knows the ShardId, ArgentSea will populate the ShardId value from this configuration data. If you provide a ShardId mapping (and include the shardid argument in the `MapShardKey` attribute), ArgentSea will understand that you want the data value instead.

### The ShardChild

The ShardChild object supports table compound keys in your sharded data. In this sample application, the Customer __Location__ records are identified by a compound key including both a CustomerId and LocationId.

## [SQL Server](#tab/tabid-sql)

```csharp
[MapShardChild('L', "CustomerId", "LocationId")]
[MapToSqlInt("CustomerId")]
[MapToSqlSmallInt("LocationId")]
public ShardChild CustomerLocationKey { get; set; }
```

## [PostgreSQL](#tab/tabid-pg)

```csharp
[MapShardChild('L', "customerid", "locationid")]
[MapToPgInteger("customerid")]
[MapToPgSmallint("locationid")]
public ShardChild CustomerLocationKey { get; set; }
```

***

It would be possible for ArgentSea to include a “ShardGrandChild” struct, for three-column compound keys, (or even a “ShardGreatGrandChild”) but the need for this hasn’t arisen.

## Loading the Shard Service

Loading and injecting the ArgentSea ShardSets service is similar to the Databases services explained in the last tutorial. However, because the ShardSets’ shard id type is generic, it can get tiresome to repeatedly include the generic type. We can simplify using the ShardSet type by declaring a child instance which defines the shard id type.

## [SQL Server](#tab/tabid-sql)

```csharp
public class ShardSets : SqlShardSets<byte>
{
    public ShardSets(
        IOptions<SqlShardConnectionOptions<byte>> configOptions,
        IOptions<SqlGlobalPropertiesOptions> globalOptions,
        ILogger<ShardSets> logger
    ) : base(configOptions, globalOptions, logger)
    {
        //
    }
}
```

Calling the ArgentSea `AddSqlService<>` (note the generic argument) extension method will load both the ArgentSea sharding and database services, *except* the ShardSet. The `AddSqlService` method does not automatically load the `SqlShardSets<>` service in order to to allow a locally defined, typed ShardSets collection to be loaded instead.

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddSqlServices<byte>(this.Configuration);
    services.AddSingleton<ShardSets>(); //Load locally defined service.
}
```

If you prefer to simply use the extant `ArgentSea.SqlShardSets` collection, you can load that instead:

```csharp
services.AddSingleton<SqlShardSets<byte>>(); //Load ArgentSea.ShardSets with generic parameter.
```

## [PostgreSQL](#tab/tabid-pg)

```csharp
public class ShardSets : PgShardSets<byte>
{
    public ShardSets(
        IOptions<SqlShardConnectionOptions<short>> configOptions,
        IOptions<SqlGlobalPropertiesOptions> globalOptions,
        ILogger<ShardSets> logger
    ) : base(configOptions, globalOptions, logger)
    {
        //
    }
}
```

Calling the ArgentSea `AddPgService<>` (note the generic argument) extension method will load both the ArgentSea sharding and database services, *except* the ShardSet. To allow a locally defined, typed ShardSets collection to be a service, you have to load it separately.

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddPgServices<byte>(this.Configuration);
    services.AddSingleton<ShardSets>(); //Load locally defined service.
}
```

If you prefer to simply use the extant `ArgentSea.PgShardSets` collection, you can load that instead:

```csharp
services.AddSingleton<PgShardSets<short>>(); //Load ArgentSea.ShardSets with generic parameter.
```

***

> [!NOTE]
> The `ShardSets` services implicitly loads the `Databases` service also. Of course, if no databases are configured, the collection will be empty.

## Queries and Data

Our implementation adds two static classes which help describe our data. The first is a simple list of constants, which correspond to the “origin” parameter of `ShardKey` and `ShardChild` objects. the “Origin” helps prevent accidental use of, say, an __Inventory__ key to delete an __Order__ record. By using constants, you cam more explicitly distinguish the “c” used for __Contact__ data from the “C” used for __Customer__ data, which might otherwise be confusing.

The second static class consolodates query definitions, as described in the [Creating SQL Queries](/querying/sql.md) tutorial. This serves two purposes: first, it becomes easy to determine which queries are actually used by the code (on large projects, this can be difficult). Also, the optional parameter list can limit the parameters that are set when Model attributes have more parameters than the query requires.

The code for these are in the `DataOrigins` and `Queries` classes respectively. The code is self explainitory.

## The Repository Pattern

Our sample code uses a `CustomerStore` class, which implements all of the actual data access logic. The theory of the repository pattern is help contain the coupling between the data layer and application logic. We could theorectically replace the data store by only changing the `CustomerStore` implementation. Because this is a web service that provides data access, the virtue of this approach is not compelling in our sample. Your milage may vary. 

Because our web service does very little *except* read and write data, there is actually very little for the controller to do. You will find nearly all of the ArgentSea implemetation in the `CustomerStore` class.

> [!NOTE]
> The code in the `CustomerStore` would have been even simpler if there was no relationship between __Customers__ and __Contacts__. The sample is intended to illustrate handling a challenging sharding scenario, but without this relationship the code would have been half as long.

This class is injected into the controller, so it need to be a registered as a service at startup.

```csharp
services.AddTransient<Stores.CustomerStore>();
```

The injectable `CustomerStore` class in turn uses the injected `ShardSets` service. Obtaining the injectible `ShardSets` service in your repository class is straightforward:

```csharp
public class CustomerStore
{
    private readonly ShardSets.ShardSet _shardSet;
    private readonly ILogger<CustomerStore> _logger;

    public CustomerStore(ShardSets shardSets, ILogger<CustomerStore> logger)
    {
        _shardSet = shardSets["Customers"];
        _logger = logger;
    }
    ....
```

If you elected to use the ArgentSea shard sets collection instead, things are just a little more verbose:

## [SQL Server](#tab/tabid-sql)

```csharp
public class CustomerStore
{
    private readonly SqlShardSets<byte>.ShardSet _shardSet;
    private readonly ILogger<CustomerStore> _logger;

    public CustomerStore(SqlShardSets<byte> shardSets, ILogger<CustomerStore> logger)
    {
        _shardSet = shardSets["Customers"];
        _logger = logger;
    }
    ....
```

## [PostgreSQL](#tab/tabid-pg)

```csharp
public class CustomerStore
{
    private readonly PgShardSets<short>.ShardSet _shardSet;
    private readonly ILogger<CustomerStore> _logger;

    public CustomerStore(PgShardSets<short> shardSets, ILogger<CustomerStore> logger)
    {
        _shardSet = shardSets["Customers"];
        _logger = logger;
    }
    ....
```

***

Using a `ShardSet` instance, you generally simply provide a query, set parameters, and invoke a method. You can query across all shards, some shards, or within a single shard instance. Querying across all or some shards can return all results in a unified list or the first valid result.

The code necessary to query all shards is very simple:

## [SQL Server](#tab/tabid-sql)

```csharp
public async Task<CustomerModel> FindByLogonName(string loginName, CancellationToken cancellation)
{
    var prms = new ParameterCollection()
        .AddSqlNVarCharInputParameter("@LoginName", loginName, 255)
        .CreateOutputParameters<CustomerModel>(_logger);
    return await _shardSet.ReadFirst.MapOutputAsync<CustomerModel, LocationModel, ContactListItem>(Queries.CustomerFind, prms, cancellation);
}
```

## [PostgreSQL](#tab/tabid-pg)

```csharp
```csharp
public async Task<CustomerModel> FindByLogonName(string loginName, CancellationToken cancellation)
{
    var prms = new ParameterCollection()
        .AddPgVarcharInputParameter("loginname", loginName, 255)
        .CreateOutputParameters<CustomerModel>(_logger);
    return await _shardSet.ReadFirst.MapOutputAsync<CustomerModel, LocationModel, ContactListItem>(Queries.CustomerFind, prms, cancellation);
}
```

***

### Optimizing Multi-Record Saves

The principal means of optimizing data access is to *limit the number of round-trips to the database server*. If the need to save ten records generates ten distinct calls to the database, the solution will not be very efficient. Unfortuantely, there is no standard way of handling multi-record saves; fortunately, there are platform-specific ways of managing it. 

## [SQL Server](#tab/tabid-sql)

SQL Server uses Table Valued Parameters to save multiple records. Our sample data saves a __Customer__ with multiple __Locations__ and multiple __Contacts__. These are both passed to Table Valued Parameters. This also allows the related records to be managed in a single internally-managed (low oeverhead) transaction. 

Table Valued Parameters require a “User Defined Type”, which defines the column names and types for each row. ArgentSea offers a SQL Server-specific Mapper which converts the metadata attrributes to the correct format for this paramteter. (Like its siblings, it also uses expression trees to to compile a high-performance solution when it is initially run). In our sample implementation, the combined mapping attributes created more columns than the User Defined Type required. To solve this, we can simply provide a list or array of names; then, only those are used.


```csharp
var prms = new ParameterCollection()
    .CreateInputParameters(customer, _logger)
    .AddSqlTableValuedParameter("@Locations", customer.Locations, customerLocationTypeColumns, _logger)
    .AddSqlTableValuedParameter<ContactListItem, byte, int>("@Contacts", customer.Contacts, "ShardId", System.Data.SqlDbType.TinyInt, "RecordId", System.Data.SqlDbType.Int);
```

In this example, there are two `AddSqlTableValuedParameter` overloads, the first uses a Model and maps to the Unser Defined Type using mapping attributes; the second example maps only the key values, using the column names supplied.

You can learn more in the section on [Multi-Record Saves](/querying/multirecord.md). 

## [PostgreSQL](#tab/tabid-pg)

PostgreSQL uses the COPY statement to quickly load multiple rows into tables or temporary tables. Once these records are loaded, a SQL statement can be run to process them further.

This process uses ArgentSea’s Batch functionality, which allows multiple steps to execute within a single open and transacted connection. In our sample, the first two steps load the __Contacts__ and __Locations__ data into temporary tables, then the third batch steps runs a SQL statement to save this data.

If the table name has a “.” schema seperater, then the target is assumed to a a standard table; without a “.” in the table name, the table is understood to be a temporary table. If the table does not exist it will be created (unless, of course, the client does not have permission to create a table). The table will have all of the coloumns defined by the model’s metadata attributes.

```csharp
var customerPrms = new ParameterCollection()
    .CreateInputParameters<CustomerModel>(customer, _logger);
var shardBatch = new ShardBatch<short, List<short>>()
    .Add(customer.Contacts, "temp_contacts")
    .Add(customer.Locations, "temp_locations")
    .Add(Queries.CustomerSave, customerPrms, "contactshardid");
```

***

## Using Swagger

when you launch the web API project, it will open to thw Swagger UI by default. If you are creating a new project, open project properties, go to the Debug tab, then change the *Launch browser:* text value to “swagger”.

The first GET method returns all customers, across all shards (you don't want to do this in the real world). 

You can selected any ShardKey in the resulting list and provide that to the other GET method, which takes a ShardKey string argument. This methods returns a complex JSON result with extended customer detail.

You can edit the customer detail and provide that to the PATCH method to update the database values.

The POST method allows you to create a new customer. If successful, it returns the ShardKey of the created record. To create a new customer you can provide the following JSON:

```json
{
  "name": "New Customer",
  "type": "WalkIn",
  "locations": [
    {
      "type": "RetailStore",
      "streetAddress": "123 Main Street",
      "locality": "Chicago",
      "region": "IL",
      "postalCode": "60612",
      "iso3166": "us",
      "coordinates": {
        "latitude": 41.867789,
        "longitude": -87.675839
      }
    },
    {
      "type": "RetailStore",
      "streetAddress": "456 Oak Avenue",
      "locality": "Dallas",
      "region": "TX",
      "postalCode": "75211",
      "iso3166": "us",
      "coordinates": {
        "latitude": 32.730430,
        "longitude": -87.675839
      }
    }
  ],
  "contacts": [
    {
        "origin": "c",
        "shardId": 1,
        "recordId": 7
    },
    {
        "origin": "c",
        "shardId": 2,
        "recordId": 4
    },
    {
        "origin": "c",
        "shardId": 1,
        "recordId": 8
    }
  ]
}
```

Finally, you can DELETE a customer by providing a ShardKey.