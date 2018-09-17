# ArgentSea Documentation

## Getting Started

If you want to understand everything first, explore the [deep dives](#deep-dives); if you prefer to learn by getting your hands dirty, jump into the [walkthroughs](#walkthroughs).

## Overview

In essence, ArgentSea data library consists of four areas of functionality. These are complementary but independent enough to be able to use some without the others:

* __Configuring connections__ - is particularly useful with the large number of connections necessary for sharded data.
* __Querying data__ - in a way that supports multi-threaded concurrent queries on multiple shards.
* __Mapping__ - simplifies and dramatically shortens the code required to map data (result sets and/or parameters) to or from .NET objects.
* __Tooling__ - a simple UI to generate starter model classes and/or CRUD procedures based on a table or view. A time saver for generating boilerplate code.

> [!TIP]
> Because ArgentSea enhances the familiar ADO.NET data access classes, you can still use ADO.NET to resolve any capability gaps or distinctive query requirements.

## Deep Dives

1. Installing ArgentSea (coming soon).
2. [Setting up your configuration](tutorials/configuration.md)
3. [Mapping](tutorials/mapping.md)
4. [Using shards](tutorials/sharding.md)

## Walkthroughs

1. [QuickStart 1](tutorials/quickstart1.md) - Setting up and configuring an initial project
2. [QuickStart 2](tutorials/quickstart2.md) - Adding shard handling and queries

## Reference

You can find the most information in the [APIsection](/reference/apis.html).
