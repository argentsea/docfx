# ArgentSea Setup

## Nuget Package

The first, obvious step is to add a reference to the relevant ArgentSea nuget package. 
These are provider-specific. Current there are two:

* ArgentSea for SQL Server
* ArgentSea for PostgreSQL

Both packages include a reference to the base ArgentSea shared package, 
so it is not necessary to include that separately, though you may need to referece it
in your code.

You *may* be able to include multiple provider packages in your progress (this is not 
a tested scenario), but you cannot have a single class that includes provider
attributes from different providers. If you need to reference different database providers 
(i.e. both SQL Server and PostgreSQL), it would be best to use different projects.


## Simplifying Object References
Of course, the ArgentSea framework does not know in advance the data types used in your 
architecture. You provide this information using generics when you compile your project.
Since the ArgentSea objects are used frequently, however, this can be unnecessarily verbose.

To simplify accessing the ArgentSea objects in your code, we suggest either 
* Creating a derived class that declares your data types
* Or adding a *using* statement in each data access class

### Using Derived classes

#### Using using


