# Environment

ArgentSea will likely be useful even if you are not using data sharding, but its genesis and continuing purpose is to facilitate high-performance data access via sharding. 

Data sharding is very difficult to retroactively add to an existing project:

* Identifying unique records on sharded data requires either very careful management of identity ranges or a “virtual compound key” (ArgentSea has classes to help with the latter approach). Either approach requires smart logic to dispatch the query to the correct shard.
* Simultaneous querying across multiple shards requires querying on multiple threads. Both ADO.NET and Entity Framework require “non-standard” implementations to enable queries on multiple threads.

Consequently, ArgentSea adds the most value if you are building a new project. And if you are creating a new project, you should use .NET Core. While much of ArgentSea may work with the legacy .NET framework, it is not tested for this purpose, and it uses services — such as dependency injection, configuration, and logging — that are implemented differently in the legacy .NET framework.

## Getting Started

