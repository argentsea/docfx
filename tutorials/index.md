# Overview

ArgentSea is built on top of ADO.NET, so an understanding of basic .NET data access is essential to understanding ArgentSea. Once you understand the architecture of the framework, you will find it no more difficult to use (and generally less difficult) than other .NET data access approaches. You can always still use ADO.NET to resolve any capability gaps or distinctive query requirements.

## [Setup Tutorial](setup.md)

Because data sharding is difficult to retroactively add to an existing project without rewriting it, if you are creating a new project, it makes sense to use .NET Core rather than then legacy .NET Framework.

ArgentSea would probably work in a legacy .NET Framework application (i.e. it is .NET standard compatible), but it is not tested for this purpose and it uses services — such as dependency injection, configuration, and logging — that are implemented differently in the different .NET versions. If you *do* use the legacy .NET Framework, please help build out our documentation with any guidance you can share.

## [Configuration Tutorial](configuration/configuration.md)

Especially when sharding is introduced, applications may need to manage *many* database connections. ArgentSea introduces a unique “hereditary configuration hierarchy”, which allows a large number of connections to be managed painlessly. This approach also allows sensitive elements — like database login passwords — to be stored securely, away from your source code. This innate flexibility also dramatically simplifies managing configuration in different dev, test, staging, and production environments.

ArgentSea fully supports scale-out deployments where writes are sent to a principal node and read activity is sent to an actively cloned endpoint. This leverages capabilities like [SQL Availability Group Readable Secondaries](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/overview-of-always-on-availability-groups-sql-server?view=sql-server-2017#ActiveSecondaries) and [SQL Read-scale Availability Groups](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/read-scale-availability-groups?view=sql-server-2017), [PostgreSQL Hot Standbys](https://www.postgresql.org/docs/11/hot-standby.html), [SQL Azure Read Scale-out](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-read-scale-out) or [SQL Azure Geo-replicas](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-geo-replication-overview), and [Amazon RDS Read Replicas](https://aws.amazon.com/rds/details/read-replicas/) or [Amazon Aurora Reader Endpoints](https://aws.amazon.com/about-aws/whats-new/2016/09/reader-end-point-for-amazon-aurora/).

ArgentSea uses the new configuration infrastructure in .NET Core, including strongly typed Options classes and injectable services, and retains the flexibility of using the many available configuration providers: JSON file, environment variables, key stores, and more.

## [Mapping Tutorial](mapping/mapping.md)

Native data access can require a lot of boilerplate code, but libraries that try to reduce this often come with performance tradeoffs. ArgentSea contains a unique ORM (Object-Relational Mapper) that is focused on parameters and procedures/functions rather than generating dynamic SQL.

Like most ORMs, simple property attributes provide metadata which makes the coding easy and simple. Unlike many ORMs, ArgentSea removes the reflection overhead by *compiling* the property-to-data mapping; subsequent mapping invocations are as fast as optimized ADO.NET code.

The Mapper flexibly handles nulls and Enumerated values. Database null values map to/from nullable types (or, occasionally, empty types). Enum properties can map to either string data fields (containing the Enum name) or numeric fields (containing the base value). The Mapper even handles nullable Enums.

## [Querying Tutorial](querying/querying.md)

The ArgentSea framework allows querying either Databases and ShardSets. When using the Mapper, this can take as little as one line of code. You may occasionally need to write a custom data handler, but this isn’t more difficult than the ADO.NET code you are used to.

Concurrent querying across multiple shards requires “non-standard” implementations of ADO.NET or Entity Framework, but ArgentSea makes this easy. If you are using shards, you can easily and simultaneously query across all shards, combining the results or getting only the first valid result.

## [Sharding Tutorial](sharding/sharding.md)

Sharded data presents two special challenges: uniquely identifying a specific record across all servers and managing data relationships between shards. Because foreign key relationships and unique constraints are no longer enforced by the database engine, a sharded application needs a global strategy for identifying records. ArgentSea offers a “virtual compound key” where the shard identifier and the record identity combine as the record key. (Of course, ArgentSea can also work with other record identity approaches too).

ArgentSea makes querying a shard set as simple as querying a single database.

Next: [Setting up](setup.md)