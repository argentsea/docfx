# Configuration Deep-Dive

## Introduction

The many environments required by SDLC processes — and possibly several Geo-dispersed production instances too — require application deployments in many distinct environments; managing configurations in each environment is already a challenge. Worse, sharded data sets can create a *very large* number of client connections, amplifying the configuration problem further. Then, scale-out of read and write endpoints doubles the number of connections. In the end, there can be a *lot* of connections to manage.

ArgentSea is designed to make this potentially large number connections manageable. Using the configuration architecture in .NET core and a unique *Hereditary Configuration Hierarchy*, ArgentSea allows application changes to be promoted through staging environments and deployed into multiple production environments. It does this while storing passwords securely and without the need for messy transformations.

## ArgentSea Data Connections

There are two types of data connections in ArgentSea:

* __A *database connection*__ - a data set which is hosted by a single database
* __A *shard set*__  - a data set spread over multiple database connections

A shard set represents a set of data that is spread among multiple database servers. This structure is common for high-performance data access, since it is usually more cost effective and predictably scalable to have multiple smaller database servers than to build one massive server. Global applications might try to improve performance for their global users by distributing shards in datacenters around the globe. The ArgentSea data access components allow you to query across multiple servers or a find specific record on its corresponding host server.

ArgentSea configuration supports any number of database definitions in the [Databases collection](https://docs.argentsea.com/api/ArgentSea.DatabasesBase-1.html), and any number of [shard sets](https://docs.argentsea.com/api/ArgentSea.ShardSetsBase-2.DataConnection.html) in the [ShardSets](https://docs.argentsea.com/api/ArgentSea.ShardSetsBase-2.html) collection. Each [shard set](https://docs.argentsea.com/api/ArgentSea.ShardSetsBase-2.DataConnection.html) can have any number of database connections (shard instances).

<table border="0" margin="0" padding="0"><tr><td width="50%"><img src="/images/databases.svg"></td><td width="50%"><img src="/images/shardsets.svg"></td></tr></table>

All data connections have the option of separate read and write connections. If you are scaling-out your data access by sharding your data, you are likely also scaling-out by separating read activity from write operations. Examples of this include [SQL Server Availability Groups](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/active-secondaries-readable-secondary-replicas-always-on-availability-groups), [PostgreSQL Hot Standby](https://www.postgresql.org/docs/11/hot-standby.html),[Amazon RDS Read Replicas](https://aws.amazon.com/rds/details/read-replicas/), [Azure SQL Geo-Replication](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-active-geo-replication), [Amazon Aurora Low-Latency Read Replicas](https://aws.amazon.com/rds/aurora/details/postgresql-details/), etc.

This creates a potentially large number of connections. Many of these will likely have similar connection information. For example, all of the connections in a shard set might use the same login information or database name, varying only the server address. To manage this redundancy, ArgentSea offers a unique *Hereditary Configuration Hierarchy*.

Next: [The Hereditary Configuration Hierarchy](hierarchy.md)
