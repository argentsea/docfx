# Logging

You have surely noticed that every Mapper command requires a logger instance — an object that implements the `ILogger` interface. A supportable application requires logging, so the parameter is not optional. The .NET Core environment provides objects that log to the console, debug window, Windows event logs, file system, [Application Insights](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-asp-net-core), [CloudWatch](https://github.com/aws/aws-logging-dotnet#aspnet-core-logging), and much more. ArgentSea can consume any of these logging providers and provide diagnostic and runtime data to their respective targets.

In production, you will generally want to use log level *Information*. In development you may find *Debug* or even *Trace* very helpful.

> [!CAUTION]
> Be sure to manage the logging level in your configuration. This determines the amount of logging and this can have a substantial impact upon performance.

## Logging Levels

The logging levels determine the types of events that are logged. These are described below:

## Critical

Logs when the circuit breaker is triggered on a connection or command. This may generate many downstream errors until the functionality is restored.

### Error

In most cases, an error condition will throw to the caller so they become the caller’s responsibility to handle or log. Because data access may happen on multiple threads, however, a simple throw may lose context. If the data reader passed to the Mapper is closed or null, this is logged as an exception along with the connection description.

## Warning

ArgentSea creates a warning log event when starting an automatic retry on a connection or command.

### Information

When the circuit breaker is triggered, ArgentSea creates a log record each time a test transaction is attempted and again when functionality is restored.

### Debug

The logged events in the Debug level are intended to help diagnose internal processes that may not be returning the expected results.

The first type of event is when a DbNull value is presented to an object that then becomes null or empty, which happens with ShardKey, ShardChild, or any object with a Required argument set to true. When this happens unexpectedly, it can be difficult to determine which database value caused the problem (as now *no* properties exist to determine the culprit). This logging event identifies which DbNull caused the result to be null or Empty.

The second type of event provides full visibility into the generated code used to build the Mapper’s activity. The Expression Tree is walked and the pseudo-code saved to the log before it is compiled. This can be extremely useful in understanding the complexities of the Mapping behavior. The log record will be rather long and the extraction may not be efficient, but it also runs only during the first data access.

This log level also reports when a parameter attribute was defined but the parameter was not found among the output parameters. This might be by design or it might be a programming oversight.

Finally, the Mapper logs when it did not find an cached delegate so an Expression Tree is being built and compiled. This is normal at startup because the cache will be empty; if these event occur afterward, there is likely a problem.

### Trace

The Mapper creates a trace log record as it iterates over properties. This can provide insight into the current context when other error conditions occur. Also, the logger will report the execution time for commands sent to a database connection or shard sets.

Next: [Querying](../querying/querying.md)
