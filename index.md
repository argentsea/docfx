![ArgentSea](/images/ArgentSeaTitle.jpg)

# ArgentSea Documentation

Modern web applications need to be built for performance and scalability, as well as security, monitoring, and configuration. ArgentSea offers a framework that consistently represents best practices for all of these concerns.

The goal of ArgentSea is to simplify the development process for delivering highly scalable and supportable applications.

## Massive Scalability

The essential ingredients for building a service that can scale to any demand include highly efficient code, reducing server round-trips, and the scale-out of reads and writes.

Highly scalable data typically means data “[sharding](tutorials/sharding/sharding.md)” — the practice of spreading data across many database servers. [Data sharding](tutorials/sharding/sharding.md) offers the most cost effective way to scale your data application as demand grows. To scale your application globally, data sharding offers the ability locate copies of your data across regional datacenters, so that data is located closer to your customers.

ArgentSea uses explicit read and write connections to enable further scale-out. By directing read activity to a mirror or cloned data set, the data load can be spread among multiple servers. Examples include [SQL Server Availability Groups](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/active-secondaries-readable-secondary-replicas-always-on-availability-groups), [PostgreSQL Hot Standby](https://www.postgresql.org/docs/11/hot-standby.html), [Amazon RDS Read Replicas](https://aws.amazon.com/rds/details/read-replicas/), [Azure SQL Geo-Replication](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-active-geo-replication), [Amazon Aurora Low-Latency Read Replicas](https://aws.amazon.com/rds/aurora/details/postgresql-details/), etc.

ArgentSea helps deliver highly optimized data access through data-to-object mapping without the overhead of reflection. The consistent use of stored procedures (SQL Server) or prepared statements (PostgreSQL) reduces both SQL compilation overhead and support/maintenance costs. Because ArgentSea can handle multiple results from the same query, the number of server round-trips can be reduced — a huge performance win.

While the genesis of ArgentSea was to support the complex requirements of data sharding, it will likely be useful for high-performance data access even if you are not accessing sharded data. Especially with a cloud infrastructure, more efficient code requires fewer resources and this translates into ongoing cost savings.

## Mission Critical Supportability

ArgentSea also addresses production concerns with built-in features like monitoring/logging, automatic retries after failures, controlling cascading failures (circuit breaking), security best-practices, and an elegant approach to managing connection configuration.

The data framework will attempt to recover from transient errors by automatically retrying the data access; you have control over how long and how often. If repeated failures occur, the system will “circuit break”, so that data failures have less chance of bringing down the whole application.

The robust logging implementation allows you to log to any provider, including [Application Insights](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-asp-net-core), [CloudWatch](https://github.com/aws/aws-logging-dotnet#aspnet-core-logging), the file system, Windows event logs, and more.

Database passwords can be secured using [Key Vault](https://azure.microsoft.com/en-us/services/key-vault/), [Secrets Manager](https://aws.amazon.com/secrets-manager/), [User Secrets](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets), [Docker secrets](https://docs.docker.com/engine/swarm/secrets/), or other secure storage.

The configuration architecture simplifies the management of large numbers of data connections, reducing redundancy while making it easy to deploy a release though staging environments.

## Maintainability via Code Clarity

Supportability is about more than managing the operational burden. It also includes simplicity in understanding application behavior, ease in extending it with new features and requirements, and a discoverable path to resolving bugs. When long-lived applications must be supported by teams that are not the original authors, this becomes especially critical.

The ArgentSea Mapper helps reduce the burden of code maintenance by simplifying and reducing data access code. This makes the code easier to understand and therefore easier to enhance and maintain. Further, the framework helps consolidate and segregate the SQL data access statements so that they can be more easily managed and optimized.

The logging functionality can also provide substantial insight to developers, including the dynamic code compilation behind the Mapper, misconfigurations, and data errors.

## Getting Started

Explore the [deep dives](tutorials/index.md) to understand the logic and services of ArgentSea. An ArgentSea implementation consists of the [NuGet library packages](tutorials/setup.md), loading the [configuration and services](tutorials/configuration/configuration.md) at startup, decorating the models classes with [data attributes](tutorials/mapping/mapping.md), and calling the various [query methods](tutorials/querying/querying.md).

If you prefer to learn by getting your hands dirty, jump into the [walkthroughs](tutorials/quickstarst/configuration.md). You can find the most detailed information in the [API section](/reference/apis.html).

Next: [Explore ArgentSea’s functionality](tutorials/index.md)
