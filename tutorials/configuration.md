# Configuring ArgentSea

## Introduction

ArgentSea fully leverages the configuration architecture of .NET core/.NET standard. If this architecture is new to you, it essentially consists of two parts:

* A configuration dictionary, which can be loaded from multiple sources, one of which is a file called *appsettings.json*
* An “options” architecture, which casts the configuration entries into a strongly-typed configuration object.

One of the key improvements of the configuration architecture in .NET standard is the dictionary architecture, which allows entries to be loaded from multiple sources. So, for example, you might load the account names from an *appsettings.json* configuration file, the passwords from a usersecrets.json file (or Key Vault), and the server names from
environment variables. Properly managed, this can make deployments both easier and more secure.

## ArgentSea Database Connections

There are two types of database connections in ArgentSea:

* __A *database connection*__ - a data set which is hosted by a single database
* __A *shard set*__  - a single data set spread over multiple database connections

ArgentSea configuration supports any number of *database connections* and any number of *shard sets*. And of course each *shard set* can have many database connections.

### Limiting Redundancy Across Multiple Connection Definitions

This creates a potentially large number of database connections. Many of these will likely have similar connection information. In many scenarios, all of the connections in a shard set would use the same login information. Likewise, in a given datacenter environment it only makes sense that all connections use the same resiliency strategy.

To manage this redundancy, the ArgentSea configuration data is broken into four parts:

* Login credential information, which can be referenced by any connection.
* Data resilience strategies, any of which can be referenced by any connection.
* Database connection information
* Shard set connection information

#### Credentials

If you are using json configuration files to manage your configuration, the credentials section in your configuration might look like this:

````json
  "Credentials": [
    {
      "SecurityKey": "0",
      "UserName": "webuser",
      "Password": "123456"
    },
    {
      "SecurityKey": "1",
      "WindowsAuth": true,
    },
    {
      "SecurityKey": "2",
      "UserName": "admin",
      "Password": "7890"
    }
  ]
````

If you prefer to set the properties of an Options class directly, you can use the ArgentSea.DataSecurityOptions class.

You should put this configuration section into a secure location. In a development environment, you should consider using the *UserSecrets* functionality, which prevents this information from being checked into your source code repository. In other environments, you might consider using you should use AWS Secrets Manager, Azure Key Vault, or something similar.

The *SecurityKey* property must be unique and exactly match the security string key that you specify on your connection (i.e. both must have the same casing).

#### Resilience Strategies

Resilience strategies define how ArgentSea recovers from unexpected failures, usually through some combination of retry logic and circuit breaking. Because one typically requires only a few resilience strategies across datacenters (perhaps one for local connections and another for across the WAN), to reduce redundancy we use the same keyed approach as for security.

A general Resilience Strategy is implicit. If a connection does not specify a Resilience Strategy, this
default one will be used. If it is defined, the corresponding connection(s) must specify the key (again, casing matters).

An example resiliency configuration section might look like this:

````json
  "ResilienceStrategies": [
    {
      "DataResilienceKey": "local",
      "RetryCount": "6",
      "RetryInterval": "150",
      "RetryLengthening": "Linear",
      "CircuitBreakerFailureCount": "10",
      "CircuitBreakerTestInterval": "5000"
    },
    {
      "DataResilienceKey": "remote",
      "RetryCount": "6",
      "RetryInterval": "250",
      "RetryLengthening": "Fibonacci",
      "CircuitBreakerFailureCount": "20",
      "CircuitBreakerTestInterval": "5000"
    }
````

##### Retries

Not that retries only occur on errors that are defined as *transient*.
A permissions error or invalid object reference would be pointless to retry.
(The list of errors defined as *transient* is in the provider-specific implementation
of IDataProviderServiceFactory. You can view this in the source code).

The `RetryCount` setting determines how many times the connection retries before aborting
and raising an error back to the caller.
The `RetryInterval` determines the length of time (in milliseconds) between retries.
The `RetryLengthening` value can add an additional pause between subsequent retries.

One might presume that if the system encounters a transient error, it should retry quickly.
Then, if the retry is not successful, it should wait a bit longer for the error to clear before
retrying again. The `RetryLengthening1 value is what determines how much longer it will pause
on subsequent retries before giving up.

The `RetryLengthening` values are:

* __Linear__ - each retry is the same duration as specified in `RetryInterval`
* __Fibonacci__ - The first retry is at `RetryInterval`, each subsequent retry interval pauses for the duration of the previous two combined.
* __HalfSquare__ - the retry count number is squared, then divided by two, then multiplied by `RetryInterval`
* __Squaring__ - each retry attempt doubles the duration of the previous one.

You can visualize the impact of `RetryLengthening` with these charts:

![Linear](../images/retrygraphs/linear.jpg)

![HalfSquare](../images/retrygraphs/halfsquare.jpg)

![Fibonacci](../images/retrygraphs/fibonacci.jpg)

![Doubling](../images/retrygraphs/doubling.jpg)

If a Resilience Strategy is not defined, ArgentSea will use a default strategy. Currently, this a is `RetryCount` of 6, `RetryInterval` of 250 milliseconds, and a
`RetryLengthening` of Fibonacci. With these values, the default resilience strategy would take a total of five seconds to finally fail.

Note that a high `RetryCount` could create a very long delay before a connection is allowed to ultimately fail.

##### Circuit Breaking

When a database connection is unavailable, this can cause serious downstream problems. Processes may pile-on further requests even while earlier requests are simply waiting
to time out. As this continues, the queue of backlogged requests becomes so large that the caller itself can manage no more. This bottleneck can block other systems too. What started as a broken connection to a single database eventually becomes fatal to the calling system too!

This is the reason to add a “circuit breaker” — a fail-fast mechanism to ensure that callers do not wait needlessly for queued connections that are unlikely to succeed, and which are blocking other processes too.

When the circuit breaker is tripped, subsequent connections will fail *immediately*. This prevents queuing, bottleneck blocking, and downstream failures. While tripped, the circuit breaker will periodically allow a single transaction to proceed; if it successful the circuit breaker is reopened. In this way, a system restoration will automatically close the circuit breaker too so that connections can resume.

The `CircuitBreakerFailureCount` value determines how many sequential failures will trigger the circuit breaker. The `CircuitBreakerTestInterval` value determines
how often (in milliseconds) the circuit breaker will allow a single transaction through.

### Database Connections

The database configuration architecure allow any number of database connections. Each connection is identified by a key, which you also use to request the connection in your code.

The connection information is specific to the database provider.

## [SQL Server](#tab/tabid-sql)

### SQL Server Database Connections

For SQL Server, the entire set of attributes would look like this:

````json

"SqlDbConnections": [
  {
    "DatabaseKey": "MyDb",
    "DataConnection": {
      "SecurityKey": "0",
      "DataResilienceKey": "remote",
      "ApplicationIntent": "ReadWrite",
      "ApplicationName": "MyWebApp",
      "ConnectRetryCount": 0,
      "ConnectRetryInterval": 0,
      "ConnectTimeout": 2,
      "CurrentLanguage": "english",
      "DataSource": "localhost",
      "Encrypt": false,
      "FailoverPartner": "",
      "InitialCatalog": "MyDb",
      "LoadBalanceTimeout": 0,
      "MaxPoolSize": 100,
      "MinPoolSize": 0,
      "MultipleActiveResultSets": false,
      "MultiSubnetFailover": true,
      "PacketSize": 8000,
      "PersistSecurityInfo": false,
      "Pooling": true,
      "Replication": false,
      "TrustServerCertificate": true,
      "TypeSystemVersion": "Latest",
      "WorkstationID": ""
    }
  }
]
````

You do *not* need include all of these attributes in your connection! Any value not included in your configuration will be set to the provider default — except as described in the next paragraphs.

The `ConnectRetryCount`, `ConnectRetryInterval` values default to 0 because the ArgentSea retry logic duplicates this functionality. If you prefer to use the SqlClient retry functionality instead, set these to their desired values and specify a `ResilienceStrategy` with no retries. If you use both connection retries *and* ArgentSea retries, no harm will come, other than a lot of retries.

The other exception to the provider default values is the `ConnectTimeout` value. The provider default is 15 seconds, but with the ArgentSea’s retry logic, this could create
unnecessarily long connection timeouts. The ArgentSea default is 2 seconds because datacenter connections are easily resolved in that time unless something is wrong.
If you have a WAN or high-latency connection (or are using ConnectRetryCount), you should consider increasing this value.

If you accept the defaults, the only required parameter values are:

````json
"SqlDbConnections": [
  {
    "DatabaseKey": 1,
    "DataConnection ": {
      "SecurityKey": "2",
      "DataResilienceKey": "remote",
      "DataSource": "localhost",
      "InitialCatalog": "MyDb",
    }
  }
]
````

## [PostgreSQL](#tab/tabid-pg)

### PostgreSQL Database Connections

For SQL Server, the entire set of attributes would look like this:

````json
"PgDbConnections": [
  {
    "DatabaseKey": "MyDB",
    "DataConnection": {
      "SecurityKey": "MyCredentials",
      "ResilienceKey": "local",
      "ApplicationName": "MyWebApp",
      "AutoPrepareMinUsages": 5,
      "CheckCertificateRevocation": false,
      "ClientEncoding": "UTF8",
      "CommandTimeout": 15,
      "ConnectionIdleLifetime": 300,
      "ConnectionPruningInterval": 10,
      "ConvertInfinityDateTime": false,
      "Database": "MyDB",
      "Encoding": "UTF8",
      "Enlist": true,
      "Host": "10.10.1.22",
      "IncludeRealm": false,
      "InternalCommandTimeout": -1,
      "KeepAlive": 0,
      "KerberosServiceName": "postgres",
      "MaxAutoPrepare": 0,
      "MaxPoolSize": 100,
      "MinPoolSize": 1,
      "NoResetOnClose": false,
      "PersistSecurityInfo": true,
      "Pooling": true,
      "Port": 5432,
      "ReadBufferSize": 8192,
      "SearchPath": "",
      "ServerCompatibilityMode": "Redshift",
      "SocketReceiveBufferSize": 8192,
      "SocketSendBufferSize": 8192,
      "SslMode": "Disable",
      "TcpKeepAliveInterval": 0,
      "TcpKeepAliveTime": 0,
      "Timeout": 2,
      "TrustServerCertificate": false,
      "UsePerfCounters": false,
      "UseSslStream": false,
      "WriteBufferSize": 8192
    }
  }
]
````

You do *not* need include all of these attributes in your connection! Any value not included in your configuration will be set to the provider default — except as described in the next paragraph.

The principal change to the provider default values is the `ConnectTimeout` value. The provider default is 15 seconds, but with the ArgentSea’s retry logic, this could create
unnecessarily long connection timeouts. The ArgentSea default is 2 seconds because datacenter connections are easily resolved in that time unless something is wrong.
If you have a WAN or high-latency connection (or are using ConnectRetryCount), you should consider increasing this value.

If you accept the defaults and are running on the default port (5432), the only required parameter values are:

````json
"PgDbConnections": [
  {
    "DatabaseKey": "MyDB",
    "DataConnection": {
      "SecurityKey": "MyCredentials",
      "Host": "localhost",
      "Database": "MyDb",
    }
  }
]

````

## Shard Set Connections

A shard set represents a single data set that is spread among multiple database servers. This is a common practice for high-performance data access, since it is usually more cost effective and predictably scalable to have multiple smaller database servers than one massive server. By locating the sharded servers in datacenters around the globe, you may optimize performance for your global customers. Using the ArgentSea data access components, you can query across multiple servers or a find specific record on its corresponding host server.

From a configuration perspective, a single sharded data introduces three concerns:

* There can be a large number of database connections to manage.
* In high-performance scenarios (especially when globally distributed), queries may *read* from a cloned database, but *write* to a different (possibly remote) database.
* The data type of the shard identifier is important because a record in a data shard may refer to records in other shards. Persisting the remote shard reference means saving the shard identifier too.

ArgentSea does support querying multiple shard sets, but you cannot join or combine records between them without implementing that code yourself. You could have a distinct shard set for, say, all of your subscriber information and a separate shard set for all of your transactions. You define the shard set name in your configuration; when you query a shard set, you specify the shard set name.

Each database in a shard set has a shard identifier. This shard identifier is used in combination with the record identifier to uniquely tag a record. In other words, records in the shard set are identified with a sort of virtual compound key, consisting of the shard identifier and the record identifier.


customer records
must be persisted along with the remote record ID. This makes the data type of the shard identifier in important consideration

A record that references another record that is located in a different shard needs special care. ArgentSea offers a “Shard Key” type as a record identifier (which we think is less cumbersome than managing distinct identity ranges on every shard). Configuration must also be aware of the nature of this shard key.


````json
"SqlShardSets": [
  {
    "ShardSetKey": "Set1",
    "Shards": [
      {
        "ShardId": 0,
        "ReadConnection": {
          "SecurityKey": "0",
          "DataResilienceKey": "local",
          "DataSource": "MyOtherServer",
          "InitialCatalog": "dbName2"
        },
        "WriteConnection": {
          "SecurityKey": "0",
          "DataResilienceKey": "local",
          "DataSource": "MyOtherServer",
          "InitialCatalog": "dbName2"
        }
      },
      {
        "ShardId": 1,
        "ReadConnection": {
          "SecurityKey": "0",
          "DataResilienceKey": "local",
          "DataSource": "MyOtherServer",
          "InitialCatalog": "dbName2"
        },
        "WriteConnection": {
          "SecurityKey": "0",
          "DataResilienceKey": "local",
          "DataSource": "MyOtherServer",
          "InitialCatalog": "dbName2"
        }
      }
    ]
  }
]
````

## MISC NOTES

Sharding introduces substantial performance benefits but also substantial complexity. Because of this complexity it is difficult to retrofit a sharding architecture onto an existing data model. Leveraging the ArgentSea framework allows you to build Shard awareness into your application.

Sharded data may need to refer to related records which reside in a different shard. In ArgentSea, this is resolved by a “ShardKey” structure. This is essentially a virtual compound key, where one part is the ShardId. Applications build with Shard awareness will likely need to store both the ShardId and record key for any related records that span shards. To maximize performance and storage flexibility, ArgentSea uses a generic value, which means you can use strings, any size of integer, or other values.

## Identifying a Record in a Sharded Data Source

In a traditional database, a record refers to another record with its “key”, usually a number. The database engine can ensure that keys are unique within a data set. With sharded data, different database engines are involved, so this...

In a sharded data source, it become difficult to ensure that a

You can better understand the *shard set* configuration by being aware of these key features:

* You can You can use any value type
* You can query across multiple shards
* You can store data geographically local
