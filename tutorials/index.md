# Overview

The genesis of ArgentSea was to support the complex requirements of data sharding, but it will likely be useful for high-performance data access even if you are not accessing sharded data.

These deep-dive tutorials explain each area in depth. If you’d prefer to learn by being hands-on, you can explore [QuickStart 1](quickstart1.md), which guides you through a project setup, and [QuickStart 2](quickstart2.md), which extends that to sharding.

## [Setup](setup.md)

The ArgentSea documentation assumes that your project is essentially new. Data sharding is very difficult to retroactively add to an existing project without rewriting it. 

And it you are creating a new project, it certainly makes sense to use .NET Core/Standard (rather than then legacy .NET Framework). ArgentSea would probably work in a .NET Framework application, it is not tested for this purpose, and it uses services — such as dependency injection, configuration, and logging — that are implemented differently in the legacy framework. If you *do* use the legacy .NET Framework, please help build out our documentation with any guidance you can share.

## [Configuration](configuration.md)

ArgentSea uses the new configuration infrastructure in .NET Core, including strongly typed Options classes and injectable services. The configuration information can be stored in a JSON file, environment variables, key stores, and more. Registering the ArgentSea services is easy with our extension method.

The configuration approach is intended to make it easy to manage lots of connections (including separate read and write endpoints), store credential information securely, and also capture resilience preferences (retries and circuit breaking).

You can configure database connections, shard sets, or both. You may encounter problems, however, if you combine multiple database platforms (SQL Server and PostgreSQL) in a single project; it is not a tested scenario.

## [Mapping](mapping.md)

Data access can require a lot of boilerplate code, much of which maps properties to data parameters, columns, etc. ArgentSea contains a distinctive ORM (Object Relational Mapper) that is focused on parameters and procedures/functions rather than generating dynamic SQL.

The mapping metadata is set by property attributes, which sets the SQL data type and name. The ORM compiles the property to data mapping during the first call; afterward, the mapping should as fast as if it had been a hand-coded ADO.NET mapping. The Mapper also allows nullable types to map the database Null values. Enum properties can map to string data fields (containing the Enum name) or numeric fields (containing the base value). The Mapper even handles nullable Enums.

## [Querying](querying.md)

The ArgentSea framework allows querying both Databases and ShardSets. When you can use the Mapping capabilities, this is dead simple. However, sometimes you need to write a custom data handler. Using ArgentSea isn’t much difficult than other ADO.NET code. Understanding the logic behind the architecture will make the design more transparent.

## [Sharding](sharding.md)

Sharded data presents two extra challenges: identifying specific records across all servers, and concurrently querying multiple servers for data.

Simultaneous querying across multiple shards requires querying on multiple threads. Both ADO.NET and Entity Framework require “non-standard” implementations to enable concurrent queries on distinct threads. ArgentSea makes this easy, as you can simply set parameters and get results on a ShardSet.

Because table relationships and unique constraints are no longer enforced by the database engine, a sharded application needs a global strategy for identifying records. One approach involves setting distinct identity ranges on every server, This can be tricky to manage. ArgentSea offers a “virtual compound key” where the shard identifier and the record identity combine as the record key. In most cases this might be the simpler approach. In the end, ArgentSea can work with either identity approach.
