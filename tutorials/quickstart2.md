# QuickStart Two

The [previous QuickStart](quickstart1.md) introduced configuration and mapping. This tutorial extends that information while working with a sharded data set. I also extends the mapping functionality to include list and object properties on the model class.

Sharded data introduces two complexities:

* How do I uniquely identify and locate a record, which could be on any shard?
* How do I manage data on one shard and related data on a foreign shard?

This walkthrough will illustrate how both challenges are met.

Although you can download QuickStart2 as a completed project, this walkthrough will guide you as if this is a new project. 

## Create the Project

Create a new “ASP.NET Core Web Application” project. When prompted, select the “API” project type. Once the solutions is created, open your dependencies and add the following NuGet packages.

* __ArgentSea.Sql__ or __ArgentSea.Pg__ - for SQL Sever or PostgreSQL databases respectively
* __Swashbuckle.Aspnetcore__ - for [Swagger](https://swagger.io/)] and for invoking the API without creating a client

To follow a standard convention for an MVC project, create folders for __Models__, __InputModels__, and __Stores__ (or “Repositories”).

## The Sample Data

Our sample application is going to track Customers. Customers can have multiple Locations (1:∞). Customers can also have Contacts, but the Contacts can belong to more than one Customer (∞:∞). The Contact may not exist in the same shard as the Customer.

The SQL for the sample data is found in the GitHub source repository.

* PostgreSQL - https://github.com/argentsea/quickstarts/tree/master/QuickStart2.Pg/QuickStart2.Pg/Sql
* SQL Server - https://github.com/argentsea/quickstarts/tree/master/QuickStart2.Sql/QuickStart2.Sql/Sql

To create the QuickStart data sources, first run the __ServerSetup.sql__ file. This will create four databases and two logins. The logins contain weak passwords (that are published on the Internet), so you might consider changing them; on the other hand, these will only have permission to execute procedures or functions in a specific namespace, so it’s not a big risk.

> [!NOTE]
> Conceptually, these four databases would correspond to regional data stores in the *United States*, *Brazil*, *Europe*, and *China*. In my real global application I would replicate the data from each region to each other region. Therefore, each region would have one writable data store and three readable ones. In this way, the “local” shard is writeable, the “foreign” shards are read-only. Writes to a foreign shard must be done across the WAN. Your implementation may vary.  
> Our walkthrough, however, only needs four simple databases on one server. We’ll only imagine the rest.

### The ShardId

Each of the four databases needs an identifier. This is more significant and may more fraught than it sounds. A thorough discussion of the options and impact is [here](sharding/shardkey.md). The précis is that once established, the type (and values) of the *ShardId* cannot be easily changed. The ShardId is a part of record identification, so confusion in configuration could result in data corruption.

In this sample application, the type of the ShardId for SQL Server is a `byte` (SQL TinyInt); for PostgreSQL it is a `short` (SQL SmallInt).

In *each* of the four databases, you will need to run the __DbSetup.sql__ file. This creates the schemas, tables, and procedures in the database. The first function is called `ws.ShardId`. *Before you run the SQL script*, change the return value (line 11) for each database:

| Database | Return Value |
| --- | :---: |
| CustomerShardUS | 1 |
| CustomerShardEU | 2 |
| CustomerShardBR | 3 |
| CustomerShardZH | 4 |

After the tables are created, run the SQL file specific to that database (i.e. __ShardBR.sql__, __ShardEU.sql__, __ShardUS.sql__, or __ShardZH.sql__). This will populate the tables with sample data.

At this point, we should have four database with identical stored procedures or functions. ArgentSea will only invoke stored procedures or functions.

## Configuring Connections

Because sharded data may require a large number of data connections, ArgentSea offers a more flexible way of managing this than by using connection strings. ArgentSea offers the “Hereditary Configuration Hierarchy”. This allows you to set an attribute at the parent level and all children will inherit this value, unless overwritten by a child. A more thorough discussion is [here](configuration\hierarchy.md).

In our sample application, we can use the same server or host for all connections; each shard connection only changes the database name. (In a production deployment, the configuration might be exactly backwards: the databases have identical names, but each is on a different host). We also want to use the *webWriter* user for write connections and *webReader* for read connections.

So the configuration settings looks like this (with annotations):

## [SQL Server](#tab/tabid-sql)

```json
{
  "SqlShardSets": [¹
    {
      "ShardSetName": "Customers",²
      "DataSource": ".",³
      "Write": {⁴
        "UserName": "webWriter",
        "Password": "Pwd567890",
      },
      "Read": {⁴
        "ApplicationIntent": "ReadOnly",
        "UserName": "webReader",
        "Password": "Pwd123456"
      },
      "Shards": [⁵
        {
          "ShardId": 1,⁶
          "InitialCatalog": "CustomerShardUS"⁷
        },
        {
          "ShardId": 2,⁶
          "InitialCatalog": "CustomerShardEU"⁷
        },
        {
          "ShardId": 3,⁶
          "InitialCatalog": "CustomerShardBR"⁷
        },
        {
          "ShardId": 4,⁶
          "InitialCatalog": "CustomerShardZH"⁷
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

⁴ `Read` and `Write` are peculiar, and optional, members of shard set’s “inheritance” chain, as their children are indirect. Any attributes defined in the shard set’s `Write` section apply only to write connections. Likewise, for `Read` connections. These values can be overwritten by shard or connection attributes.

⁵ `Shards` is an array of shard connections, one for each shard in the shard set.

⁶ `ShardId` is a *required* identifier for the shard. This value is essential for finding and identifying a sharded record. It cannot be duplicated within a shard set. If the type of the shard identifier is a string, then this value should have quotes around it in the JSON file.

⁷ `InitialCatalog` is a connection attribute. Because it appears at the shard level, both read connections and write connections for this shard will inherit this value.

## [PostgreSQL](#tab/tabid-pg)

```json
{
  "PgShardSets": [¹
    {
      "ShardSetName": "Customers",²
      "Host": "localhost",³
      "Write": {⁴
        "UserName": "webWriter",
        "Password": "Pwd567890"
      },
      "Read": {⁴
        "UserName": "webReader",
        "Password": "Pwd123456"
      },
      "Shards": [⁵
        {
          "ShardId": 1,⁶
          "Database": "CustomerShardUS"⁷
        },
        {
          "ShardId": 2,⁶
          "Database": "CustomerShardEU"⁷
        },
        {
          "ShardId": 3,⁶
          "Database": "CustomerShardBR"⁷
        },
        {
          "ShardId": 4,⁶
          "Database": "CustomerShardZH"⁷
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

⁴ `Read` and `Write` are peculiar, and optional, members of shard set’s “inheritance” chain, as their children are indirect. Any attributes defined in the shard set’s `Write` section apply only to write connections. Likewise, for `Read` connections. These values can be overwritten by shard or connection attributes.

⁵ `Shards` is an array of shard connections, one for each shard in the shard set.

⁶ `ShardId` is a *required* identifier for the shard. This value is essential for finding and identifying a sharded record. It cannot be duplicated within a shard set. If the type of the shard identifier is a string, then this value should have quotes around it in the JSON file.

⁷ `Database` is a connection attribute. Because it appears at the shard level, both read connections and write connections for this shard will inherit this value.

***

This hierarchy, then, defines a server name once, to be used for the entire shard set. The read and write logins are also defined once, to be used by all read or write connections in the shard set. Each shard has a distinct database name. ArgentSea can build read and write connections to each data store without the need to configure any of this data redundantly — the login, server name, and database names are each managed only once.

In your project, open the appsettings file and add this configuration, updating the JSON to the appropriate server references. 

You might consider moving the login password information to the UserSecrets store, which is a best practice. Simply remove the password entries from the appsettings.json hierarchy and add them to the usersecrets.json file.

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
> The configuration arrays in appsettings.json and usersecrets.json will not match if they do not appear in exactly same order. In this sample, we have only one shard set and the passwords are not in the `Shards` array, so this is not a problem.

## Creating the Models

