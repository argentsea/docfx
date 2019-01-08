# Loading the Configuration

ArgentSea fully leverages the configuration architecture of .NET Core. If this architecture is new to you, it essentially consists of two parts:

* A configuration *Dictionary*, which can be loaded from multiple sources — one of which is typically a file called *appsettings.json*
* An “Options” architecture, which casts the configuration entries into a strongly-typed configuration objects.

One of the key improvements of the configuration architecture in .NET standard is the dictionary architecture, which allows entries to be loaded and combined from multiple sources. So, for example, you might load the account names from an *appsettings.json* configuration file, the passwords from a *secrets.json* file (or [Key Vault](https://azure.microsoft.com/en-us/services/key-vault/) or [Secrets Manager](https://aws.amazon.com/secrets-manager/)), and the server names from environment variables. Properly managed, this can make deployments both easier and more secure.

The previous examples all used JSON for configuration. This is not a requirement. In non-JSON configuration contexts, like environment variables, you can specify these values as key-value pairs. The key concatenates the hierarchy separated by “:”. Arrays and lists should include an index.

The same configuration information listed above can be stored and loaded as key-value pairs. An example of the same shard set configuration in non-JSON format would be:

## [SQL Server](#tab/tabid-sql)

| Key | Value |
| --- | --- |
| SqlShardSets:0:ShardSetName | Primary |
| SqlShardSets:0:DataSource | DbServer1 |
| SqlShardSets:0:FailoverPartner | Mirror1 |
| SqlShardSets:0:UserName | webUser |
| SqlShardSets:0:Password | pwd1234 |
| SqlShardSets:0:Shards:0:ShardId | 0 |
| SqlShardSets:0:Shards:0:InitialCatalog | ShardDb1 |
| SqlShardSets:0:Shards:0:ReadConnection:ApplicationIntent | ReadOnly |
| SqlShardSets:0:Shards:0:ReadConnection:DataSource | Mirror1 |
| SqlShardSets:0:Shards:1:ShardId | 1 |
| SqlShardSets:0:Shards:1:InitialCatalog | ShardDb2 |
| SqlShardSets:0:Shards:1:ReadConnection:ApplicationIntent | ReadOnly |
| SqlShardSets:0:Shards:1:ReadConnection:DataSource | Mirror1 |

## [PostgreSQL](#tab/tabid-pg)

| Key | Value |
| --- | --- |
| PgShardSets:0:ShardSetName | Primary |
| PgShardSets:0:DataSource | DbServer1 |
| PgShardSets:0:UserName | webUser |
| PgShardSets:0:Password | pwd1234 |
| PgShardSets:0:Shards:0:ShardId | 0 |
| PgShardSets:0:Shards:0:InitialCatalog | ShardDb1 |
| PgShardSets:0:Shards:0:ReadConnection:DataSource | HotStandby1 |
| PgShardSets:0:Shards:1:ShardId | 1 |
| PgShardSets:0:Shards:1:InitialCatalog | ShardDb2 |
| PgShardSets:0:Shards:1:ReadConnection:DataSource | HotStandby1 |

***

Because .NET Core’s configuration architecture allows values to be aggregated from multiple data stores, we can create configuration entries where they can be most conveniently managed. Gone are the days of needing to transform configuration files upon deployment (unless you really like that type of thing).

Typically, there are three types of configuration entries:

* Secure values, like passwords or keys, which should not be readily accessible.
* Environment-specific values, like server names, which change as releases are promoted through various environments.
* Application values, which specify how the application should consistently behave.

This capability is critical for managing configuration outside of the application. Using key-value pairs allows configuration values to be hosted in environment variables and secure stores.

### Securing Passwords

In a development environment, you should consider using the [UserSecrets](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets) functionality, which prevents this information from being checked into your source code repository.

UserSecrets uses a JSON file, so the password entries can be simply removed from the appsettings.json file and stored in the *secrets.json* file. Note that the object counts must be consistent between your *appsettings.json* file and the *secrets.json* file.

## [SQL Server](#tab/tabid-sql)

### User Secrets

````json
{
  "SqlDbConnections": [
    {
    },
    {
    },
    {
      "ReadConnection": {
        "Password": "pwd1234"
      },
      "WriteConnection": {
        "Password": "pwd5678"
      }
    }
  ]
}
````

## [PostgreSQL](#tab/tabid-pg)

### User Secrets

````json
  "PgDbConnections": [
    {
    },
    {
    },
    {
      "ReadConnection": {
        "Password": "pwd1234"
      },
      "WriteConnection": {
        "Password": "pwd5678"
      }
    }
  ]
}
````

***

In other environments, you might consider using [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/), [Azure Key Vault](https://azure.microsoft.com/en-us/services/key-vault/), [Docker secrets](https://docs.docker.com/engine/swarm/secrets/), a secure file share, or something similar.

Most of these secrets stores store key value pairs. The .NET configuration manager resolves the hierarchical JSON files with a semi-colon, “:”, separator between the property hierarchy. Array elements are referenced with an index value.

> [!TIP]
> Systems that don’t support semi-colon separators in their keys (AWS Secrets Manager, for example) can use double underscores (“__”) instead.

Consequently, the previous JSON values could be saved as corresponding key-value pairs as:

## [SQL Server](#tab/tabid-sql)

| Key | Value |
| --- | --- |
| SqlDbConnections:2:ReadConnection:Password | pwd1234 |
| SqlDbConnections:2:WriteConnection:Password | pwd5678 |

## [PostgreSQL](#tab/tabid-pg)

| Key | Value |
| --- | --- |
| PgDbConnections:2:ReadConnection:Password | pwd1234 |
| PgDbConnections:2:WriteConnection:Password | pwd5678 |

***

In reality, most implementations would have a single secure password used for every shard connection, in which case the key in your secrets store is simply:

## [SQL Server](#tab/tabid-sql)

| Key | Value |
| --- | --- |
| SqlShardSets:0:Password | pwd1234 |

## [PostgreSQL](#tab/tabid-pg)

| Key | Value |
| --- | --- |
| PgShardSets:0:Password | pwd1234 |

***

### Environment-specific Configuration

Managing configuration through multiple staging and release environments works the same way. You can store environment-specific settings — like server names or database names — in server [environment variables](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-2.1#environment-variables-configuration-provider). These are also key-value pairs, so the .NET Core configuration hierarchy would be rendered to keys in the same way as described in the previous section.

The database configuration JSON presented previously would be saved in environment variables as:

| Key | Value |
| --- | --- |
| SqlDbConnections:0:DataSource | DbServer1 |
| SqlDbConnections:0:InitialCatalog | MainDb |
| SqlDbConnections:1:DataSource | DbServer1 |
| SqlDbConnections:1:InitialCatalog | OtherDb |
| SqlDbConnections:1:WriteConnection:DataSource | DbServer2 |
| SqlDbConnections:1:ReadConnection:DataSource | DbServer3 |
| SqlDbConnections:1:ReadConnection:InitialCatalog | Db1 |
| SqlDbConnections:1:WriteConnection:DataSource | DbServer4 |
| SqlDbConnections:1:WriteConnection:InitialCatalog | Db2 |

> [!NOTE]
> The order of configuration loading matters. Your Startup class should load *appsettings.json* first. When the environment variables are subsequently loaded, any existing values in *appsettings.json* will be overwritten.

ArgentSea uses the configuration dictionary built into .NET Core, so you can use any compatible configuration provider — including files, command line arguments, databases, and [more](https://rimdev.io/speech-recognition-configuration-provider-for-asp.net-core/).

There are no restrictions on which configuration entries belong to which providers (data sources). You can store passwords in environment variables or even command arguments, if you want to. Pick the right platform to management your data effectively, ArgentSea can use it as long as the values are consolidated correctly into the configuration dictionary.

### Configuration *Options*

The next phase in .NET Core’s configuration processing is the Options rendering. This converts the aggregated dictionary entries into strongly typed configuration objects, called Options. ArgentSea uses these typed Options objects to build its ShardSets and Databases services.

Unfortunately, a misconfigured property or entry can cause the Options class to be null when a value was expected. This can be difficult to debug, as there are often no error messages, just a null result. You can debug the Options classes during startup to see which values are unexpectedly null. Experiment with removing configuration values until the Options classes render as expected.

The JSON object hierarchy and property types should exactly match those of the Options objects, so if you have any doubts, explore the Options classes using C# or other strongly typed language.

## Loading the Configuration

ArgentSea uses .NET Core’s built-in Options configuration and dependency injection architecture. The complexity of turning a JSON configuration file into a connection object is as simple as calling an extension method in the `ConfigureServices` method of your `Startup` class.

## [SQL Server](#tab/tabid-sql)

This example assumes that your ShardId type is *byte*. If you use any other type, change the generic parameter.

If you use ArgentSea database connections *without* sharding, simply remove the generic declaration altogether (i.e. `services.AddPgServices(Configuration);` only).

````csharp
        public void ConfigureServices(IServiceCollection services)
        {
            ...
            // add your injectable logging provider
            services.AddLogging();
            // add the ArgentSea SQL database connections (ShardId type: byte)
            services.AddSqlServices<byte>(Configuration);
            // now add your custom data classes, which use the data components
            services.AddSingleton<MyDataStore>();
            ...
            services.AddMvc();
            ...
        }

````

## [PostgreSQL](#tab/tabid-pg)

This example assumes that your ShardId type is *short*. If you use any other type, change the generic parameter.

If you use ArgentSea database connections *without* sharding, simply remove the generic declaration altogether (i.e. `services.AddPgServices(Configuration);` only).

````csharp
        public void ConfigureServices(IServiceCollection services)
        {
            ...
            // add your injectable logging provider
            services.AddLogging();
            // add the ArgentSea SQL database connections (ShardId type: short)
            services.AddPgServices<short>(Configuration);
            // now add your custom data classes, which use the data components
            services.AddSingleton<MyDataStore>();
            ...
            services.AddMvc();
            ...
        }
        public IConfiguration Configuration { get; }

````

***

This code references a `Configuration` property. It is common practice to obtain the configuration object from the constructor of the `Startup` class, then use this to set the `Configuration` property.
