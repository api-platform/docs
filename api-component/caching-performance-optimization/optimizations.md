# Optimizations and profiling

Depending on your model architecture, several optimizations can be easily achieved to improve performance.

## Enabling the Metadata Cache

Computing metadata used by the bundle is a costly operation. Fortunately, metadata can be computed once and then cached.
API Platform internally uses a [PSR-6](https://www.php-fig.org/psr/psr-6/) cache. If the Symfony Cache component is available
(the default in the API Platform distribution), it automatically enables support for the best cache adapter available.

Best performance is achieved using [APCu](https://github.com/krakjoe/apcu). Be sure to have the APCu extension installed
on your production server. API Platform will automatically use it.

For more information about Symfony's built-in cache mechanism, please refer [to its documentation](https://symfony.com/doc/current/cache.html#configuring-cache-with-frameworkbundle).

## Doctrine Queries and Indexes

### Search Filter

When using the `SearchFilter` and case insensitivity, Doctrine will use the `LOWER` SQL function. Depending on your
driver, you may want to carefully index it by using a [function-based
index](https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search) or it will impact performance
with a huge collection. [Here are some examples to index LIKE
filters](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/like-performance-tuning) depending on your
database driver.

### Eager Loading

By default Doctrine comes with [lazy loading](https://www.doctrine-project.org/projects/doctrine-orm/en/latest/reference/working-with-objects.html#by-lazy-loading) - usually a killer time-saving feature but also a performance killer with large applications.

Fortunately, Doctrine offers another approach to solve this problem: [eager loading](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/working-with-objects.html#by-eager-loading).
This can easily be enabled for a relation: `@ORM\ManyToOne(fetch="EAGER")`.

By default in API Platform, we made the choice to force eager loading for all relations, with or without the Doctrine
`fetch` attribute. Thanks to the eager loading [extension](../fetching-and-persisting-data/extensions.md). The `EagerLoadingExtension` will join every
readable association according to the serialization context. If you want to fetch an association that is not serializable,
you have to bypass `readable` and `readableLink` by using the `fetchEager` attribute on the property declaration, for example:

```php
/**
 * @ApiProperty(attributes={"fetchEager": true})
 */
 public $foo;
```

#### Max Joins

There is a default restriction with this feature. We allow up to 30 joins per query. Beyond that, an
`ApiPlatform\Core\Exception\RuntimeException` exception will be thrown but this value can easily be increased with a
bit of configuration:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    eager_loading:
        max_joins: 100
```

Be careful when you exceed this limit, it's often caused by the result of a circular reference. [Serializer groups](../serialization/index.md)
can be a good solution to fix this issue.

#### Force Eager

As mentioned above, by default we force eager loading for all relations. This behaviour can be modified in the
configuration in order to apply it only on join relations having the `EAGER` fetch mode:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    eager_loading:
        force_eager: false
```

#### Override at Resource and Operation Level

When eager loading is enabled, whatever the status of the `force_eager` parameter, you can easily override it directly
from the configuration of each resource. You can do this at the resource level, at the operation level, or both:

```php
<?php
// api/src/Entity/Address.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource
 * @ORM\Entity
 */
class Address
{
    // ...
}
```

```php
<?php
// api/src/Entity/User.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource(attributes={"force_eager"=false})
 * @ORM\Entity
 */
class User
{
    /**
     * @var Address
     *
     * @ORM\ManyToOne(targetEntity="Address", fetch="EAGER")
     */
    public $address;

    /**
     * @var Group[]
     *
     * @ORM\ManyToMany(targetEntity="Group", inversedBy="users")
     * @ORM\JoinTable(name="users_groups")
     */
    public $groups;

    // ...
}
```

```php
<?php
// api/src/Entity/Group.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource(
 *     attributes={"force_eager"=false},
 *     itemOperations={
 *         "get"={"force_eager"=true},
 *         "post"
 *     },
 *     collectionOperations={
 *         "get"={"force_eager"=true},
 *         "post"
 *     }
 * )
 * @ORM\Entity
 */
class Group
{
    /**
     * @var User[]
     *
     * @ManyToMany(targetEntity="User", mappedBy="groups")
     */
    public $users;

    // ...
}
```

Be careful, the operation level is higher priority than the resource level but both are higher priority than the global
configuration.

#### Disable Eager Loading

If for any reason you don't want the eager loading feature, you can turn it off in the configuration:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    eager_loading:
        enabled: false
```

The whole configuration described before will no longer work and Doctrine will recover its default behavior.

### Partial Pagination

When using the default pagination, the Doctrine paginator will execute a `COUNT` query on the collection. The result of the
`COUNT` query is used to compute the last page available. With big collections this can lead to quite long response times.
If you don't mind not having the last page available, you can enable partial pagination and avoid the `COUNT` query:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    collection:
        pagination:
            partial: true # Disabled by default
```

More details are available on the [pagination documentation](../pagination-filters-sorting/pagination.md#partial-pagination).

## Using PPM (PHP-PM)

Response time of the API can be improved up to 15x by using [PHP Process Manager](https://github.com/php-pm/php-pm). If
you want to use it on your project, follow the documentation dedicated to Symfony on the PPM website.

Keep in mind that PPM is still in an early stage of development and can cause issues in production.


## Profiling with Blackfire.io

Blackfire.io allows you to monitor the performance of your applications. For more information, visit the [Blackfire.io website](https://blackfire.io/).

To configure Blackfire.io follow these simple steps:

1. Add the following to your `docker-compose.yml` file

```yaml
    blackfire:
        image: blackfire/blackfire
        environment:
            # Exposes the host BLACKFIRE_SERVER_ID and TOKEN environment variables.
            - BLACKFIRE_SERVER_ID
            - BLACKFIRE_SERVER_TOKEN
```

2. Add your Blackfire.io id and server token to your `.env` file at the root of your project (be sure not to commit this to a public repository)

        BLACKFIRE_SERVER_ID=xxxxxxxxxx
        BLACKFIRE_SERVER_TOKEN=xxxxxxxxxx

    or set it in the console before running Docker commands

        $ export BLACKFIRE_SERVER_ID=xxxxxxxxxx
        $ export BLACKFIRE_SERVER_TOKEN=xxxxxxxxxx

3. Install and configure the Blackfire probe in the app container, by adding the following to your `./Dockerfile`

```dockerfile
        RUN version=$(php -r "echo PHP_MAJOR_VERSION.PHP_MINOR_VERSION;") \
            && curl -A "Docker" -o /tmp/blackfire-probe.tar.gz -D - -L -s https://blackfire.io/api/v1/releases/probe/php/alpine/amd64/$version \
            && mkdir -p /tmp/blackfire \
            && tar zxpf /tmp/blackfire-probe.tar.gz -C /tmp/blackfire \                        
            && mv /tmp/blackfire/blackfire-*.so $(php -r "echo ini_get('extension_dir');")/blackfire.so \
            && printf "extension=blackfire.so\nblackfire.agent_socket=tcp://blackfire:8707\n" > $PHP_INI_DIR/conf.d/blackfire.ini
```

4. Rebuild and restart all your containers

        $ docker-compose build
        $ docker-compose up -d

For details on how to perform profiling, see [the Blackfire.io documentation](https://blackfire.io/docs/integrations/docker#using-the-client-for-http-profiling).
