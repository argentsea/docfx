# Configuring ArgentSea

## Discussion
ArgentSea fully leverages the configuration architecture of .NET core/.NET standard. 
In essence, this new configuration architecture consists of two parts:
* A configuration dictionary, which can be loaded from multiple
sources, including *appsettings.json*
* An “options” architecture, which casts the configuration entries into a configuration object.

### Configuration Entries
One of the key improvements of the configuration architecture in .NET standard is the 
dictionary architecture, which allows entries to be loaded from multple sources. So, 
for example, you might load the account names from an *appsettings.json* configuration file, 
the passwords from a usersecrets.json file (or Key Vault), and the server names from 
environment variables. Property managed, this can make deployments both easier and more secure.


### Limiting Redundancy with Multiple Connections
There are two types of database connections in ArgentSea, a *database connection* (a data set 
which is hosted by a single database) and a *shard set* (i.e. a single data set spread 
over multiple database connections)

ArgentSea configuration supports any number of database connections and any number of shard sets. 
And each shard set can have many database connections.

This creates a potentially large number of database connections. Many of these will likely
have similar connection information. In many scenarios, all of the
connections in a shard set would use the same login information. Likewise, in a given datacenter 
environment it only makes sense that all connections use the same resiliency strategy.

To manage this redundancy, the ArgentSea configuration data is broken into four parts.
* Login credential information, which can be referenced by any connection.
* Data resilience strategies, any of which can be referenced by any connection.
* Database connection information
* Shard set connection information

#### Credentials
The credentials section in your configuration might look like this:
````
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
If you prefer to set an Options class, you can use the ArgentSea.DataSecurityOptions class.
 
You should put all (or at least the password portion) of this section into a secure location. 
In a development environment, you should consider using the *UserSecrets* functionaly, which
prevent this information from being checked into your source code repository. In other environments, you might consider using 
you should use a secure share or Key Vault.

The SecurityKey property must be unique and exactly match the security string key that you 
specify on your connection.

#### Resilience Strategies
Likewise, because one typically require only a few resilience strategies across datacenters, 
we use the same approach to reduce redundancy. 

A connection is not required to specify a Resilience Strategy, but if it is defined, 
it must exactly match the key defined in this section. If no Resilience Stratgy is 
defined, a default one is used.

An example resiliency configuration section might look like this:
````
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
      "RetryLengthening": "Finonacci",
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
Then, if the retry is not sucessful, it should wait a bit longer for the error to clear before 
retrying again. The `RetryLengthening1 value is what determines how much longer it will pause
on subsequent retries before giving up.

The `RetryLengthening` values are:

* __Linear__ - each retry is the same duration as specified in `RetryInterval`
* __Fibonacci__ - The first retry is at `RetryInterval`, each subsequent retry interval pauses for the duration of the previous two combined.
* __HalfSquare__ - the retry count number is squared, then divided by two, then multipled by `RetryInterval`
* __Squaring__ - each retry attempt doubles the duration of the prevous one.

You can visualize the impact of `RetyLengthing` with these charts:

![Linear](../images/retrygraphs/linear.jpg)

![HalfSquare](../images/retrygraphs/halfsquare.jpg)

![Fibonacci](../images/retrygraphs/fibonacci.jpg)

![Doubling](../images/retrygraphs/doubling.jpg)


If a Resilience Strategy is not defined, ArgentSea will use a default strategy. 
Currently, this a is `RetryCount` of 6, `RetryInterval` of 250 milliseconds, and a 
`RetryLengthening` of Fibonacci. In total, the default resilience strategy would take five seconds to finally 
finally fail.

Note that a high `RetryCount` could create a very long delay before a 
connection is allowed to ultimately fail.

##### Circuit Breaking
When a database connection is unavailable, this can cause serious downstream problems.
Processes may pile-on further requests even while eariler requests are simply waiting
to time out. As this continues, the queue of backlogged requests becomes so large that the 
caller itself can manage no more. What started as a broken connection to a single database 
eventually becomes fatal to the calling system too!

This is the reason to add a “circuit breaker” — a fail-fast mechanism
to ensure that callers do not wait needless for queued connections that are unlikely to 
succeed.

When a connection repeatedly fails, the circuit breaker causes subsequent connections 
to fail immediately. The `CircuitBreakerFailureCount` value determines how many sequential
failures will trigger the circuit breaker.

Of course, we hope our transient error is resolved quickly, so we want to periodically test
whether the problem is resolved. The `CircuitBreakerTestInterval` value determines
how often (in milliseconds) the circuit breaker will allow a single transaction through.

If the single transaction succeeds, the circuit breaker reopens and transactions again resume
as normal.

### Database Connections
The database configuration architecure allow any number of database connections. Each connection
is identified by a key, which you also use to request the connection.

The connection information is specific to the database provider. 

#### SQL Server Connections
For SQL Server, the entire set of attributes would look like this:
````
"SqlDbConnections": [
  {
    "DbConnectionId": 1,
    "DbConnection": {
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

You do *not* need include all of these attributes in your connection! Any value not included 
in your configuration will be set to the provider default, except as described in the next 
paragraphs.

The `ConnectRetryCount`, `ConnectRetryInterval` values default to 0 because the ArgentSea
retry logic duplicates this functionality. If you prefer to use the SqlClient retry functionality
instead, set these to their desired values and specify a `ResilienceStrategy` with no retries. 
If you use both connection retries *and* ArgentSea retries, no harm will come, other than a lot
of retries.

The other exception to the provder default values is the `ConnectTimeout` value. 
The provider default is 15 seconds, but with the ArgentSea’s retry logic, this could create
unnecessarily long connection timeouts. The ArgentSea default is 2 seconds because 
datacenter connections are easily resolved in that time unless something is wrong. 
If you have a WAN or high-latency connection (or are using ConnectRetryCount), 
you should consider increasing this value.

If you accept the defaults, the only required parameter values are:
````
"SqlDbConnections": [
  {
    "DbConnectionId": 1,
    "DbConnection": {
      "DataSource": "localhost",
      "InitialCatalog": "MyDb",
    }
  }
]

````




### Shard Set Connections


#### Configuring Database Connections
The full configuration values for ar



#### Configuring ShardSet Connections




### Configuration Options


## Step-by-step Walkthrough


