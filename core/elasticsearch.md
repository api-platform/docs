# Elasticsearch Support

## Overview

Elasticsearch is a distributed RESTful search and analytics engine capable of solving a growing number of use cases:
application search, security analytics, metrics, logging, etc.

API Platform comes natively with the **reading** support for Elasticsearch. It uses internally the official PHP client
for Elasticsearch: [Elasticsearch-PHP](https://www.elastic.co/guide/en/elasticsearch/client/php-api/current/index.html).

Be careful, API Platform only supports Elasticsearch >= 6.5.0.

## Enabling Reading Support

To enable the reading support for Elasticsearch, simply require the Elasticsearch-PHP package using Composer:

```console
composer require elasticsearch/elasticsearch:^6.0
```

Then, enable it inside the API Platform configuration:

```yaml
# config/packages/api_platform.yaml
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

## Creating Models

API Platform follows the best practices of Elasticsearch:

* a single index per resource should be used because Elasticsearch is going to [drop support for index types and will allow only a single type per
index](https://www.elastic.co/guide/en/elasticsearch/reference/current/removal-of-types.html);
* index name should be the short resource name in lower snake case;
* the default `_doc` type should be used;
* all fields should be lower case and should use camel case for combining words.

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
// api/src/Model/User.php
namespace App\Model;

use ApiPlatform\Core\Annotation\ApiProperty;
use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource]
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
// api/src/Model/Tweet.php
namespace App\Model;

use ApiPlatform\Core\Annotation\ApiProperty;
use ApiPlatform\Core\Annotation\ApiResource;

 #[ApiResource]
class Tweet
{
    #[ApiProperty(identifier: true)]
    public string $id = '';

    public User $author;

    public \DateTimeInterface $date;

    public string $message;
}
```

API Platform will automatically disable write operations and snake case document fields will automatically be converted to
camel case object properties during serialization.

Keep in mind that it is your responsibility to populate your Elasticsearch index. To do so, you can use [Logstash](https://www.elastic.co/products/logstash),
a custom [data persister](data-persisters.md#creating-a-custom-data-persister) or any other mechanism that suits your
project (such as an [ETL](https://en.wikipedia.org/wiki/Extract,_transform,_load)).

You're done! The API is now ready to use.

### Creating custom mapping

If you don't follow the Elasticsearch recommendations, you may want a custom mapping between API Platform resources and
Elasticsearch indexes/types.

For example, consider an index being similar to a database in an SQL database and a type being equivalent to a table.
So the `User` and `Tweet` resources of the previous example would become `user` and `tweet` types in an index named `app`:

```yaml
# config/packages/api_platform.yaml
parameters:
    # ...
    env(ELASTICSEARCH_HOST): 'http://localhost:9200'

api_platform:
    # ...
    
    mapping:
        paths: ['%kernel.project_dir%/src/Model']

    elasticsearch:
        hosts: ['%env(ELASTICSEARCH_HOST)%']
        mapping:
            App\Model\User:
                index: app
                type: user
            App\Model\Tweet:
                index: app
                type: tweet

    #...
```

## Filtering

See how to use Elasticsearch filters and how to create Elasticsearch custom filters in [the Filters chapter](filters.md).

## Creating Custom Extensions

See how to create Elasticsearch custom extensions in [the Extensions chapter](extensions.md).
