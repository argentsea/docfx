# ArgentSea Documentation

Modern web applications need to be built for performance and scalability, as well as security, monitoring, and configuration. ArgentSea offers a framework that consistently represents best practices for all of these concerns.

When you select a solution, it is important to know which problems they are intended to fix. Some data libraries are designed to make it easier for non-SQL developers to work with database data. Others are intended to make developers more productive by reducing the coding effort.

ArgentSea is primarily intended to address two problems: scalability and supportability. ArgentSea optimizes the work of your developers without compromising the performance or security of your infrastructure.

## Massive Scalability

Highly scalable data means data “sharding” — the practice of spreading data across many database servers. [Data sharding](tutorials/sharding.md) offers the most cost effective way to scale your data application as demand grows. To scale your application globally, data sharding offers the ability locate copies of your data across regional datacenters, so that data is located closer to your customers.

High-performance data access means simultaneous queries across shards, data-to-object mapping without the overhead of reflection, and the consistent use of stored procedures/functions to reduce SQL compilation overhead.

While the genesis of ArgentSea was to support the complex requirements of data sharding, it will likely be useful for high-performance data access even if you are not accessing sharded data.

## Mission Critical Supportability

ArgentSea also addresses production concerns with built-in features like monitoring/logging, automatic retries after failures, controlling cascading failures, security best-practices, and managing connection configuration.

The data framework will attempt to recover from transient errors by automatically retrying the data access; you have control over how long and how often. If repeated failures occur, the system will “circuit break”, so that data failures have less chance of bringing down the whole application. The logging implementation allows you to log to any provider, including [Application Insights](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-asp-net-core), [CloudWatch](https://github.com/aws/aws-logging-dotnet#aspnet-core-logging), the file system, Windows event logs, and more. Database passwords can be [secured](tutorials/security.md) using [Key Vault](https://azure.microsoft.com/en-us/services/key-vault/), [Secrets Manager](https://aws.amazon.com/secrets-manager/), [User Secrets](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets), or other secure storage, while connection information can be separately managed per environment.

### Code Clarity and Maintenance

Supportability is about more than operational management. It also includes simplicity in understanding application behavior, ease in extending it with new features and requirements, and a natural path to resolving bugs. When applications must be supported by teams that are not the original authors, this becomes especially critical.

The ArgentSea Mapper helps reduce the burden of code maintenance by simplifying data access code. The logging functionality can also provide substantial insight to developers, including the dynamic code compilation behind the Mapper, misconfigurations, and data errors.

However, one of the top ways that ArgentSea help with ongoing code maintenance is that it enforces the use of stored procedures or functions.

<div>
    <div style="padding-left:10px;padding-right:10px;display:flex;flex-flow:row wrap;justify-content:space-around;">
        <div style="display:flex;flex-direction:column;">
            <div style="display:flex;flex-direction:row;">
                <img style="height:75px;width:75px;" src="/images/tightly-coupled.svg" />
                <div>
                    <h4>Tight Coupling</h4>
                    <p>
                        Although it might sound cozy, “tight coupling” isn’t a good thing in software design. It happens when one system’s integration with another system depends on the internal implementation of the other system. Usually the result of haphazard design, it makes it nearly impossible to switch to a different provider.
                    </p>
                    <p>
                        Like, say, when accessing your database *depends upon how tables and columns are implemented*.
                    </p>
                </div>
            </div>
        </div>
        <div style="display:flex;width:98%;flex-direction:column;">
            <div style="display:flex;flex-direction:row;">
                <img style="height:75px;width:75px;" src="/images/loosely-coupled.svg" />
                <div>
                    <h4>Loose Coupling</h4>
                    <p>
                    “Loosely coupled” systems have well-defined interfaces. You can change the implementation as long as you maintain the interface contract. These systems are more robust, testable, and manageable — and those are good things.
                    </p><p>
                    Stored procedures or functions allow you to change the underlying database structures, but as long as the same results are returned you will not break the application.
                    </p>
                </div>
            </div>
        </div>
    </div>
</div>

Stored procedures enable *loose coupling* between the app domain and the data domain. They generally perform better and offer better security too. This is why ArgentSea was built to work with stored procedures. Because ArgentSea encourages you to do your data-domain work in SQL and your application work in .NET, both your developers and your database administrators have far better control over your data.

## Getting Started

If you like to understand everything first, explore the [deep dives](tutorials/index.md); if you prefer to learn by getting your hands dirty, jump into the [walkthroughs](#walkthroughs).

## Deep Dives

1. Installing ArgentSea (coming soon).
2. [Setting up your configuration](tutorials/configuration.md)
3. [Querying](tutorials/querying.md)
4. [Mapping](tutorials/mapping.md)
5. [Using shards](tutorials/sharding.md)

## Walkthroughs

1. [QuickStart 1](tutorials/quickstart1.md) - Setting up and configuring an initial project
2. [QuickStart 2](tutorials/quickstart2.md) - Adding shard handling and queries

## Reference

You can find the most detailed information in the [API section](/reference/apis.html).