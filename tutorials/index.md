# Introduction

In essence, ArgentSea consists of four areas of functionality, which are complementary but still rather independent (i.e. you can use one without the others):

* __Configuring connections__ - is particularly useful with the large number of connections necessary for sharded data.
* __Querying data__ - in a way that supports multi-threaded concurrent queries on multiple shards.
* __Mapping__ - simplifies and dramatically shortens the code required to map data (result sets and/or parameters) to or from objects.
* __Tooling__ - a simple UI to generate starter model classes and/or CRUD procedures based on a table or view.

Because ArgentSea enhances the familiar ADO.NET data access classes, you can still use ADO.NET to resolve any capability gaps or distinctive query requirements.

## Getting Started

If you want to understand everything first, explore the [deep dives](#deep-dives); if you prefer to learn by getting your hands dirty, jump into the [walkthroughs](#walkthroughs).

## Deep Dives

1. Installing ArgentSea.
2. [Setting up your configuration](configuration.md)
3. [Mapping](mapping.md)
4. [Using shards](sharding.md)

## Walkthroughs
1. [QuickStart 1](quickstart1.md) - Setting up and configuring an initial project
2. [QuickStart 2](quickstart2.md) - Adding shard handling and queries