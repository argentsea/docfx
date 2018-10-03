# Querying Data

This deep dive assumes that you have your project correctly [configured](configuration.md) and that you have some understanding of ArgentSea’s [Mapping](mapping.md) capabiliites

ArgentSea was originally built to support application data sharding. Even if you do not use data sharding in your application, a brief discussion of the issues will help explain the architecture behind of ArgentSea’s data access approach.

## Understanding Sharding

The best way to understand the query architecture of ArgentSea is to describe a typical ADO.NET query then describe how this must change to account for concurrent multi-threaded queries across a shard set. These changes are not complicated, and they should be helpful even if your project does use sharding.

A typical ADO.NET data access method follows these steps:

1. Start with a *connection* object (generally declared with a outer `using` statement) created from a connection string.
2. Create a *command* object, providing the *connection* object as an argument (usually with an inner `using` block).
3. Next, the command's *Parameters* property is populated with the necessary input and output parameters.
4. The command is executed on the opened connection.
5. A result object is created and its properties are populated with the data results (output parameters and/or DataReader).

In a sharded environment, however:

* The same parameters must be executed on multiple connections — reversing the steps 1 to 3.
* A distinct command object must be executed and the results processed on a separate thread for each connection. The parameters cannot be shared (different threads would overwrite each other’s values) and the result handler must be thread-safe.

To simplify, ArgentSea manages the challenges of multi-threaded access access with a four-step approach:

1. Declare the parameters and arguments that will be passed to the stored procedures.
2. Create a thread for each connection, then create the *connection*, *command*, and *parameters* objects and execute the query on each thread.
3. When each query completes, call a thread-safe result handling delegate.
4. Combine the results and return them to the caller.

Understanding this approach will help you understand the general ArgentSea paradigm: Parameters are created first, then provided to the query method. The query method gets the data and invokes a [QueryResultModelHandler](/api/ArgentSea.QueryResultModelHandler-3.html) delegate to turn the output parameters and/or DataReader results into an object result.  This ArgentSea query paradigm applies even to non-sharded queries using the Databases collection.

In other words, the [ShardSet](/api/ArgentSea.ShardSetsBase-2.ShardSet.html) manages the complexity of initializing multiple queries on multiple connections and multiple results, but it is the delegate that takes the database results (from each connection/thread) an create an object result.

> [!NOTE]
> Because the Mapper *is* a thread-safe, high-performance `QueryResultModelHandler` delegate, ArgentSea is able to use attributes to automatically convert query output to a result object. Developers able to consistently use the Mapper may be insulated from the need to write any data handlers.  
> If the Mapper is not sufficient, or if you cannot add attributes to existing objects, you can handle the query results by simply writing your own delegate.

Writing a `QueryResultModelHandler` delegate is nearly identical to writing any other ADO.NET result handling code. You have access to output parameters and the DataReader and you can construct a complex object and its properties any way you like. The delegate even has a parameter that allows you to provide custom data (through the query method) with which to construct your result object.



| Database Methods |
| --- | --- |
| ListAsync | Returns a list |
| LookupAsync | returns |
| RunAsync | returns |
| QueryAsync | returns |
| ReadAsync | returns |
| GetOutAsync | returns |

| ShardSet Methods |
| --- | --- |
| QueryAll | Returns a list |
| QueryFirst  | returns |
