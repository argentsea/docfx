# The Coding Sequence

ArgentSea was originally built to support application data sharding. Today, it offers valuable capabilities that recommend it even if you do not use data sharding in your application. A brief discussion of the issues raised with sharding will help explain the architecture behind of ArgentSea’s data access approach. Ultimately, the ArgentSea approach has a slightly different sequence, but it is no more difficult than any other ADO.NET query.

## Accommodating Sharding

The best way to understand the query architecture of ArgentSea is to describe a typical ADO.NET query then describe how this must change to account for concurrent multi-threaded queries across a shard set. To keep both practices and tooling consistent, and because it really is not complicated, this same approach is used whether or not sharding is required.

A typical ADO.NET data access method follows these steps:

1. Start with a __connection__ object, created from a connection string.
2. Create a __command__ object that is associated with the __connection__ object.
3. Next, the populate the command's *Parameters* property with the necessary input and output parameters.
4. Open the connection and run the command.
5. Create a Model object (or list) and use the __DataReader__ (or output parameters) to map each column result to each of the Model’s properties.

In a sharded environment, however, the same parameters must be executed on multiple connections — reversing the steps 1 to 3. Furthermore, a distinct command object must be executed and the results processed on a separate thread for each connection. The parameters cannot be shared (different threads would overwrite each other’s values) and the result handler must be thread-safe because it could be simultaneously executing on different connections.

ArgentSea manages the challenges of multi-threaded access with a differently ordered sequence:

1. Declare the parameters and arguments that will be passed to the stored procedures.
2. Create a thread for each shard connection, then create the __connection__ (and __command__) object for each.
3. Copy the parameter values to the parameter collection on each shard’s __command__ object.
4. Call the stored procedure/function on each shard’s thread. When results are obtained, call (thread-safe) code to create and populate a Model object.
5. Merge the results and return them to the caller.

Ultimately, using ArgentSea on multiple shards is no more difficult than writing simple ADO.NET database access code (and usually much easier), but the code new needs to be grouped and sequenced differently.

## The ArgentSea Query Paradigm

Previously, you would usually use just one data access command object, which would host the ADO.NET parameters, and run the query, converting the results to a Model object. Now, because processing results is multi-threaded whereas setting up the query is not, you need to split that process into *two* procedures:

* The __*caller*__ method sets the parameters and calls an ArgentSea query method. This executes on a single thread.
* The __*handler*__ procedure converts the results to a Model object result. This can execute on many threads.

This ArgentSea query paradigm applies even to non-sharded queries using the Databases collection. This provides some design consistency, and also enables the Mapper for both sharded and non-sharded data.

> [!TIP]
> If you use ArgentSea’s optional [Mapping](../mapping/mapping.md) functionality, the multi-threaded results handling procedure is *already provided* by the Mapper. You do not have to write a handler.

Next: [Setting Parameters](setparameters.md)
