# Overview

The genesis of ArgentSea was to support the complex requirements of data sharding, but will likely be useful for high-performance data access even if you are not accessing sharded data.

These deep-dive tutorials explain each area in depth.

## [Setup](setup.md)

The ArgentSea documentation assumes that your project is essentially new. Data sharding is very difficult to retroactively add to an existing project without rewriting it. 

And it you are creating a new project, it certainly makes sense to use .NET Core/Standard (rather than then legacy .NET Framework). ArgentSea would probably work in a .NET Framework application, it is not tested for this purpose, and it uses services — such as dependency injection, configuration, and logging — that are implemented differently in the legacy framework. If you do use the .NET Framework, please help build out our documentation with any guidance you can share.

## [Configuration](configuration.md)

ArgentSea uses the new configuration infrastructure in .NET Core, including strongly typed Options classes and injectable services. The configuration information can be stored in a JSON file, environment variables, key stores, and more. Registering the ArgentSea services is easy with an extension method.

The configuration approach is intended to make it easy to manage lots of connections (including separate read and write endpoints), store credential information securely, and also capture resilience preferences (retries and circuit breaking).

You can configure database connections, shard sets, or both. You may encounter problems, however, if you combine multiple database platforms (SQL Server and PostgreSQL) in a single project; it is not a tested scenario.

## [Mapping](mapping.md)

Data access can require a lot of boilerplate code, much of which maps properties to data parameters, columns, etc. ArgentSea contains a distinctive ORM (Object Relational Mapper) that is focused on parameters and procedures/functions rather than generating dynamic SQL. 

The mapping metadata is set by property attributes, which sets the SQL data type and name. The ORM compiles the property to data mapping during the first call; afterward, the mapping should as fast as if it had been a hand-coded ADO.NET mapping. The Mapper also allows nullable types to map the database Null values. Enum properties can map to string data fields (containing the Enum name) or numeric fields (containing the base value). The Mapper even handles nullable Enums.

## [Sharding](sharding.md)

If you are new to sharding, the key issues are:

Relationships between tables are no longer enforced or even possible. A sharded data model must now cope with related records that may reside on a different server. One helpful practice is to avoid sharding *all* of the tables, particularly the lookup tables which are essentially static. In some cases, the only solution involves sequential queries (once to get the the original record and then subsequent queries to get related rows). ArgentSea does not plan your data model for you.

Unique record identifiers can be a challenge because the database engine can no longer enforce uniqueness. A common approach involves setting distinct identity ranges on every server — which can be tricky to manage and hard to re-balance. Also, errors are easy to make and can quickly lead to a catastrophic data cleanup initiative. ArgentSea offers a “virtual compound key” where the shard identifier and the record identity combine as the record key. The problem of which connection should be used for a given record query also becomes elementary with this approach. In the end, ArgentSea can work with either identity approach.

Simultaneous querying across multiple shards requires querying on multiple threads. Both ADO.NET and Entity Framework require “non-standard” implementations to enable concurrent queries on distinct threads. ArgentSea makes this easy, as you can simply set parameters and get results on a ShardSet.