# Mapping

The Mapper make data-access coding simpler and more productive by using  property attributes to map a model class’s properties to data values — parameters, reader columns, and (in the case of SQL Server) table-value parameters. This reduces and simplifies the amount of code required to render data.

## Overview

Using the Mapper consists of two parts:

* Add attributes to a model class define how each property should be mapped to a data store (if at all)
* Call a method which maps properties to parameters and/or maps data results to properties

By defining metadata about the names of parameters or result columns, the Mapper can automatically map properties to columns and/or parameters. Several query methods on both Database connections and ShardSets implicitly use the Mapper.

## Performance

The ArgentSea Mapper is written to be as high-performance as optimized hand-coded data access code. However, there is a hitch.

Property attributes can only be retrieved using *reflection*, which is relatively slow .NET code. To avoid this type of performance penalty on every data access, ArgentSea uses reflection only the first time the mapping is performed; using that metadata it then creates and compiles an “Expression Tree”to build an optimized, compiled mapping. The compiled code is cached in memory and reused for all subsequent calls.

> [!WARNING]
> The Mapper will be relatively slow (and CPU intensive) the *first time* each model class is mapped to parameters or data. The initial compilation usually takes less than a second. Subsequent calls will execute the data to property mapping at native machine-code speeds. When the application is restarted, the memory is cleared and the compilation overhead occurs again.

## Missing Parameters or Columns

In some cases, the Model may have more properties than are defined in a parameter list or in data reader columns. When this happens, ArgentSea is somewhat forgiving.

As part of a Query definition (`QueryStatement` or `QueryProcedure` class), you can optionally specify the parameter set. If set, only parameters included in this list will be mapped from the Model object. This is a great way to suppress values not needed for this particular request. Of course, if the Mapper sets/reads/writes parameters and the query does not include that parameter, ADO.NET will return an error if the query is sent to the database.

If a data reader result does not contain an expected column, the property is simply ignored. If the logging level is “Debug” or lower, a log message will be created including the Model name and column name.

Next: [Property Attributes](attributes.md)