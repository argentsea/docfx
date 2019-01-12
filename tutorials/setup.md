# ArgentSea Setup

ArgentSea is a *.NET Standard* application, which means it should work with .NET Core, Xamarin, and the .NET Framework 4.6.1 or higher. However, ArgentSea depends on services provided by .NET Core — like logging and dependency injection — that may require “coercion” in the other environments. Because data sharding is so difficult to add to an existing application, the general assumption is that consumers would be new applications created using .Net Core. If you use another framework, please help with the documentation! (And tell us if there is demand for a NuGet package targeting your framework).

## Setup Steps

Only a few steps are necessary to use ArgentSea in your project:

* Install the appropriate NuGet package.
* Define the configuration metadata, usually in appsettings.json
* Load the configuration and injectable Services in your Startup class.
* Decorate your model classes with data attributes (optional).
* Invoke data access methods from the `Databases` or `ShardSets` services.

## NuGet

There are currently two versions of ArgentSea: one for SQL Server and the other for PostgreSQL. To include ArgentSea into your project, simply search NuGet for ArgentSea.Sql (SQL Server) or ArgentSea.Pg (PostgreSQL).

Both packages install a shared ArgentSea package (which has most of the actual code). Consequently, there are two namespaces; some objects will be in the `ArgentSea` namespace; others will be in the `ArgentSea.Sql` or `ArgentSea.Pg` namespace.

> [!NOTE]
> You *may* be able to include multiple provider packages in your project  (i.e. both PostgreSQL and SQL Server), but this is not a tested scenario.  
> You *cannot* have a single model class that includes provider attributes from different providers; this is, you can’t use the same model class to read/write to both SQL Server and PostgreSQL. If you need to reference different database providers, the practical solution would be to use different projects (microservices).

## Dependencies

ArgentSea has very few dependencies. Other than a few Microsoft packages (logging, configuration, options, and immutable collections), the only dependency is upon [Polly](http://www.thepollyproject.org). Of course, the data platform libraries each take a dependency upon their respective ADO.NET libraries, [SqlClient](https://docs.microsoft.com/en-us/dotnet/api/system.data.sqlclient?view=netcore-2.2) and [Npgsql](https://www.npgsql.org/).

Next: [Configuration](configuration/configuration.md)
