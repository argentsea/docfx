# ArgentSea Setup

## NuGet Package

The first, obvious step is to add a reference to the relevant ArgentSea NuGet package. Currently, this is not available as we are finalizing code before springing for a code signing certificate (which is necessary for publishing a NuGet package).

When this becomes available, this page will be updated.

These are currently two provider-specific packages:

* ArgentSea for SQL Server
* ArgentSea for PostgreSQL

Both packages include a reference to the base ArgentSea shared package, so it is not necessary to include that separately, though you may need to reference it in your code.

You *may* be able to include multiple provider packages in your progress (this is not a tested scenario), but you cannot have a single class that includes provider attributes from different providers. If you need to reference different database providers (i.e. both SQL Server and PostgreSQL), it would be best to use different projects.