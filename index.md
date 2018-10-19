# ArgentSea Documentation

Modern web applications need to be built for performance and scalability, as well as security, monitoring, and configuration. ArgentSea offers a framework that consistently represents best practices for all of these concerns.

Highly scalable data means data “sharding” — the practice of spreading data across many database servers. The genesis of ArgentSea was to support the complex requirements of data sharding, although it will likely be useful for high-performance data access even if you are not accessing sharded data.

To simplify development without compromising performance, ArgentSea offers a Mapper that performs as well as native code.

## Overview

ArgentSea is structured to optimize the performance of your data servers and also the performance of your data developers. It establishes a clear pattern for accessing data that minimizes the complexity of accessing both sharded and non-sharded data.

ArgentSea also addresses production concerns with built-in features like monitoring/logging, automatic retries after failures, controlling cascading failures, security, and managing connection configuration. These and other benefits may be useful even if you are not using sharded data.

Data sharding offers the most cost effective way to scale your data application as demand grows. To scale your application globally, data sharding offers the ability locate your data across regional datacenters, so that data is located closer to your customers.

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

You can find the most information in the [APIsection](/reference/apis.html).
