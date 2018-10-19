# Overview

These deep-dive tutorials explain distinct functional areas in depth. If you’d prefer to learn by being hands-on, you can explore [QuickStart 1](quickstart1.md), which guides you through a project setup, and [QuickStart 2](quickstart2.md), which extends that to sharding.

ArgentSea is built on top of ADO.NET, so an understanding of basic .NET data access is be essential to understanding ArgentSea. Once you understand the architecture of the framework, you will find it no more difficult to use (and generally less difficult) than other .NET data access approaches. You can always still use ADO.NET to resolve any capability gaps or distinctive query requirements.

## [Setup](setup.md)

Data sharding is difficult to retroactively add to an existing project without rewriting it. If you are creating a new project, it certainly makes sense to use .NET Core or .NET Standard (rather than then legacy .NET Framework).

ArgentSea would probably work in a .NET Framework application, it is not tested for this purpose, and it uses services — such as dependency injection, configuration, and logging — that are implemented differently in the legacy framework. If you *do* use the legacy .NET Framework, please help build out our documentation with any guidance you can share.

## [Configuration](configuration.md)

ArgentSea uses the new configuration infrastructure in .NET Core, including strongly typed Options classes and injectable services. The configuration information can be stored in a JSON file, environment variables, key stores, and more. Registering the ArgentSea services is easy with our extension method.

You can configure database connections, shard sets, or both. The configuration approach is intended to make it easy to manage lots of connections (including separate read and write endpoints), store credential information securely, and also capture resilience preferences (retries and circuit breaking).

You may encounter problems, however, if you combine multiple database platforms (i.e. both SQL Server and PostgreSQL) in a single project; it is not a tested scenario.

## [Mapping](mapping.md)

Data access can require a lot of boilerplate code, much of which maps properties to data parameters, columns, etc. ArgentSea contains a distinctive ORM (Object Relational Mapper) that is focused on parameters and procedures/functions rather than generating dynamic SQL.

The mapping metadata is set by property attributes. The ORM compiles the property to data mapping during the first call; afterward, the mapping should as fast as if it had been a hand-coded ADO.NET mapping.

The Mapper enables nullable types to map to database Null values. Enum properties can map to either string data fields (containing the Enum name) or numeric fields (containing the base value). The Mapper even handles nullable Enums.

## [Querying](querying.md)

The ArgentSea framework allows querying both Databases and ShardSets. When you can use the Mapping capabilities, this is dead simple. However, sometimes you need to write a custom data handler. Using ArgentSea isn’t much difficult than other ADO.NET code, once you understand the logic behind the framework.

## [Sharding](sharding.md)

Sharded data presents two extra challenges: identifying specific records across all servers, and concurrently querying multiple servers for data.

Because table relationships and unique constraints are no longer enforced by the database engine, a sharded application needs a global strategy for identifying records. ArgentSea offers a “virtual compound key” where the shard identifier and the record identity combine as the record key. ArgentSea also works with other record identity approaches.

Simultaneous querying across multiple shards requires querying on multiple threads. Both ADO.NET and Entity Framework require “non-standard” implementations to enable concurrent queries on distinct threads. ArgentSea makes this easy, as you can simply set parameters and get results on a ShardSet.