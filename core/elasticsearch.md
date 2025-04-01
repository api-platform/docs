# Elasticsearch Support

## Overview

Elasticsearch is a distributed RESTful search and analytics engine capable of solving a growing number of use cases:
application search, security analytics, metrics, logging, etc.

API Platform comes natively with the **reading** support for Elasticsearch. It uses internally the official PHP client
for Elasticsearch: [Elasticsearch-PHP](https://www.elastic.co/guide/en/elasticsearch/client/php-api/current/index.html).

Be careful, API Platform only supports Elasticsearch >= 7.11.0 < 8.0 and Elasticsearch >= 8.4 < 9.0. Support for 
Elasticsearch 8 was introduced in API Platform 3.2.

## Enabling Reading Support

To enable the reading support for Elasticsearch, simply require the Elasticsearch-PHP package using Composer. For
Elasticsearch 8:

```console
composer require elasticsearch/elasticsearch:^8.4
```

For Elasticsearch 7:
```console
composer require elasticsearch/elasticsearch:^7.11
```

Then, enable it inside the API Platform configuration, using one of the configurations below:

### Enabling Reading Support using Symfony

```yaml
# api/config/packages/api_platform.yaml
parameters:
  # ...
  env(ELASTICSEARCH_HOST): 'http://localhost:9200'

api_platform:
  # ...

  mapping:
    paths: ['%kernel.project_dir%/src/Model']

  elasticsearch:
    hosts: ['%env(ELASTICSEARCH_HOST)%']

  #...
```

### Enabling Reading Support using Laravel

```php
<?php
// config/api-platform.php
return [
    // ....
    'mapping' => [
        'paths' => [
            base_path('app/Models'),
        ],
    ],
    'elasticsearch' => [
        'hosts' => [
            env('ELASTICSEARCH_HOST', 'http://localhost:9200'),
        ],
    ],
];
```

## Creating Models

API Platform follows the best practices of Elasticsearch:

- a single index per resource should be used because Elasticsearch is going to [drop support for index types and will
allow only a single type per index](https://www.elastic.co/guide/en/elasticsearch/reference/current/removal-of-types.html);
- index name should be the short resource name in lower snake_case;
- the default `_doc` type should be used;
- all fields should be lower case and should use camelCase for combining words.

This involves having mappings and models which absolutely match each other.

Here is an example of mappings for 2 resources, `User` and `Tweet`, and their models:

`PUT user`

```json
{
  "mappings": {
    "_doc": {
      "properties": {
        "id": {
          "type": "keyword"
        },
        "gender": {
          "type": "keyword"
        },
        "age": {
          "type": "integer"
        },
        "first_name": {
          "type": "text"
        },
        "last_name": {
          "type": "text"
        },
        "tweets": {
          "type": "nested",
          "properties": {
            "id": {
              "type": "keyword"
            },
            "date": {
              "type": "date",
              "format": "yyyy-MM-dd HH:mm:ss"
            },
            "message": {
              "type": "text"
            }
          },
          "dynamic": "strict"
        }
      },
      "dynamic": "strict"
    }
  }
}
```

`PUT tweet`

```json
{
  "mappings": {
    "_doc": {
      "properties": {
        "id": {
          "type": "keyword"
        },
        "author": {
          "properties": {
            "id": {
              "type": "keyword"
            },
            "gender": {
              "type": "keyword"
            },
            "age": {
              "type": "integer"
            },
            "first_name": {
              "type": "text"
            },
            "last_name": {
              "type": "text"
            }
          },
          "dynamic": "strict"
        },
        "date": {
          "type": "date",
          "format": "yyyy-MM-dd HH:mm:ss"
        },
        "message": {
          "type": "text"
        }
      },
      "dynamic": "strict"
    }
  }
}
```

```php
<?php
// api/src/Model/User.php with Symfony or app/Model/User.php with Laravel
namespace App\Model;

use ApiPlatform\Elasticsearch\State\CollectionProvider;
use ApiPlatform\Elasticsearch\State\ItemProvider;
use ApiPlatform\Elasticsearch\State\Options;
use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;

#[ApiResource(
    operations: [
        new GetCollection(provider: CollectionProvider::class, stateOptions: new Options(index: 'user')),
        new Get(provider: ItemProvider::class, stateOptions: new Options(index: 'user')),
    ],
)]
class User
{
    #[ApiProperty(identifier: true)]
    public string $id = '';

    public string $gender;

    public int $age;

    public string $firstName;

    public string $lastName;

    /**
     * @var Tweet[]
     */
    public iterable $tweets = [];
}
```

```php
<?php
// api/src/Model/Tweet.php with Symfony or app/Model/Tweet.php with Laravel
namespace App\Model;

use ApiPlatform\Elasticsearch\State\CollectionProvider;
use ApiPlatform\Elasticsearch\State\ItemProvider;
use ApiPlatform\Elasticsearch\State\Options;
use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;

#[ApiResource(
    operations: [
        new GetCollection(provider: CollectionProvider::class, stateOptions: new Options(index: 'tweet')),
        new Get(provider: ItemProvider::class, stateOptions: new Options(index: 'tweet')),
    ],
)]
class Tweet
{
    #[ApiProperty(identifier: true)]
    public string $id = '';

    public User $author;

    public \DateTimeInterface $date;

    public string $message;
}
```

API Platform will automatically disable write operations and snake_case document fields will automatically be converted to
camelCase object properties during serialization.

Keep in mind that it is your responsibility to populate your Elasticsearch index. To do so, you can use [Logstash](https://www.elastic.co/products/logstash),
a custom [state processors](state-processors.md#creating-a-custom-state-processor) or any other mechanism that suits your
project (such as an [ETL](https://en.wikipedia.org/wiki/Extract,_transform,_load)).

You're done! The API is now ready to use.

## Filtering

See how to use Elasticsearch filters and how to create Elasticsearch custom filters in the
[Elasticsearch filters documentation](../core/elasticsearch-filters.md).

## Creating Custom Extensions

See how to create Elasticsearch custom extensions in [the Extensions chapter](extensions.md#custom-elasticsearch-extension).
