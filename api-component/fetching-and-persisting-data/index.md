# Introduction 

API-Platform has been thought to expose your model as a REST API regardless of how you store and/or query them:

* Relational databases: MySQL, MariaDB, PostgreSQL, Oracle, ...
* NoSQL databases: MongoDB, Cassandra, Redis, Couchbase, ...
* Search engines: Solr, ElasticSearch, Algolia, ...
* 3rd-party APIs, SOAP clients, proprietary SDKs, ...

## Data providers and Data persisters

To make API-Platform _"persistence-agnostic"_, we have introduced the concepts of [data providers](data-providers.md) for read operations (GET) 
and [data persisters](data-persisters.md) for write operations (POST, PATCH, PUT, DELETE).

However, as we want it to be ready to use, we already provide several implementations:

### Relational databases

API-Platform ships with a built-in persistence layer which leverages the [Doctrine project](https://www.doctrine-project.org/), 
an open-source ORM (Object Relational Mapper) which maps your model to your database using the [Data Mapper pattern](https://en.wikipedia.org/wiki/Data_mapper_pattern).

If your model is backed by a relational database, you can already enjoy standard features such as:

* Filters (value, range, date, ...)
* Pagination
* Ordering

Learn [how to configure them](../pagination-filters-sorting/index.md) and [how to create your own filters](extensions.md).

### NoSQL databases

If you use MongoDB, the [Doctrine project](https://www.doctrine-project.org/) also provides an ODM (Object Document Mapper) 
which will work similarly. 

To get started, read [how to enable MongoDB support](mongodb.md).

Other NoSQL applications (Cassandra, Couchbase, ...) are not covered by this implementation, which is MongoDB-specific. Please refer to [Other cases](#other-cases).

### Search Engines

If your master data is stored in an ElasticSearch engine, we've got you covered! 

API-Platform provides an [ElasticSearch bridge](elasticsearch.md) with built-in pagination, filters and sorting. 

The only limitation is that it doesn't support **write** operations for the moment: your persistence logic belongs to you.

### Other cases

Cases which are not covered above will require you to write your own. See the related articles:

* [Data providers](data-providers.md) for building your reading logic
* [Data persisters](data-persisters.md) for building your writing logic
