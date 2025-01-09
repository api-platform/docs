# Filters

API Platform Core provides a generic system to apply filters and sort criteria on collections.
Useful filters for Doctrine ORM, MongoDB ODM and ElasticSearch are provided with the library.

You can also create custom filters that fit your specific needs.
You can also add filtering support to your custom [data providers](data-providers.md) by implementing interfaces provided
by the library.

By default, all filters are disabled. They must be enabled explicitly.

When a filter is enabled, it automatically appears in the [OpenAPI](swagger.md) and [GraphQL](graphql.md) documentations.
It is also automatically documented as a `hydra:search` property for JSON-LD responses.

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/filters?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Filtering and Searching screencast"><br>Watch the Filtering & Searching screencast</a></p>

## Doctrine ORM and MongoDB ODM Filters

### Basic Knowledge

Filters are services (see the section on [custom filters](#creating-custom-filters)), and they can be linked
to a Resource in two ways:

1. Through the resource declaration, as the `filters` attribute.

   For example having a filter service declaration:

    ```yaml
    # config/services.yaml
    services:
        # ...
        offer.date_filter:
            parent: 'api_platform.doctrine.orm.date_filter'
            arguments: [ { dateProperty: ~ } ]
            tags:  [ 'api_platform.filter' ]
            # The following are mandatory only if a _defaults section is defined with inverted values.
            # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
            autowire: false
            autoconfigure: false
            public: false
    ```

   We're linking the filter `offer.date_filter` with the resource like this:

   <code-selector>

    ```php
    <?php
    // api/src/Entity/Offer.php

    namespace App\Entity;

    use ApiPlatform\Core\Annotation\ApiResource;

    #[ApiResource(attributes: ['filters' => ['offer.date_filter']])]
    class Offer
    {
        // ...
    }
    ```

    ```yaml
    # config/api_platform/resources.yaml
    App\Entity\Offer:
        collectionOperations:
            get:
                filters: ['offer.date_filter']
        # ...
    ```

    ```xml
    <?xml version="1.0" encoding="UTF-8" ?>
    <!-- api/config/api_platform/resources.xml -->

    <resources xmlns="https://api-platform.com/schema/metadata"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="https://api-platform.com/schema/metadata
            https://api-platform.com/schema/metadata/metadata-2.0.xsd">
        <resource class="App\Entity\Offer">
            <collectionOperations>
                <collectionOperation name="get">
                    <attribute name="filters">
                        <attribute>offer.date_filter</attribute>
                    </attribute>
                </collectionOperation>
                <!-- ... -->
            </collectionOperations>
        </resource>
    </resources>
    ```

   </code-selector>

2. By using the `#[ApiFilter]` attribute.

   This attribute automatically declares the service, and you just have to use the filter class you want:

    ```php
    <?php
    // api/src/Entity/Offer.php

    namespace App\Entity;

    use ApiPlatform\Core\Annotation\ApiFilter;
    use ApiPlatform\Core\Annotation\ApiResource;
    use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter;

    #[ApiResource]
    #[ApiFilter(DateFilter::class, properties: ['dateProperty'])]
    class Offer
    {
        // ...
    }
    ```

   Learn more on how the [ApiFilter attribute](filters.md#apifilter-attribute) works.

   For the sake of consistency, we're using the attribute in the below documentation.

   For MongoDB ODM, all the filters are in the namespace `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Filter`. The filter
   services all begin with `api_platform.doctrine_mongodb.odm`.

### Search Filter

If Doctrine ORM or MongoDB ODM support is enabled, adding filters is as easy as registering a filter service in the
`config/services.yaml` file and adding an attribute to your resource configuration.

The search filter supports `exact`, `partial`, `start`, `end`, and `word_start` matching strategies:

* `partial` strategy uses `LIKE %text%` to search for fields that contain `text`.
* `start` strategy uses `LIKE text%` to search for fields that start with `text`.
* `end` strategy uses `LIKE %text` to search for fields that end with `text`.
* `word_start` strategy uses `LIKE text% OR LIKE % text%` to search for fields that contain words starting with `text`.

Prepend the letter `i` to the filter if you want it to be case insensitive. For example `ipartial` or `iexact`. Note that
this will use the `LOWER` function and **will** impact performance [if there is no proper index](performance.md#search-filter).

Case insensitivity may already be enforced at the database level depending on the [collation](https://en.wikipedia.org/wiki/Collation)
used. If you are using MySQL, note that the commonly used `utf8_unicode_ci` collation (and its sibling `utf8mb4_unicode_ci`)
are already case-insensitive, as indicated by the `_ci` part in their names.

Note: Search filters with the `exact` strategy can have multiple values for the same property (in this case the condition will be similar to a SQL IN clause).

Syntax: `?property[]=foo&property[]=bar`

In the following example, we will see how to allow the filtering of a list of e-commerce offers:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;

#[ApiResource]
#[ApiFilter(SearchFilter::class, properties: ['id' => 'exact', 'price' => 'exact', 'description' => 'partial'])]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
    offer.search_filter:
        parent: 'api_platform.doctrine.orm.search_filter'
        arguments: [ { id: 'exact', price: 'exact', description: 'partial' } ]
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined with inverted values.
        # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
        autowire: false
        autoconfigure: false
        public: false

# config/api/Offer.yaml
App\Entity\Offer:
    # ...
    collectionOperations:
        get:
            filters: ['offer.search_filter']
```

</code-selector>

`http://localhost:8000/api/offers?price=10` will return all offers with a price being exactly `10`.
`http://localhost:8000/api/offers?description=shirt` will return all offers with a description containing the word "shirt".

Filters can be combined together: `http://localhost:8000/api/offers?price=10&description=shirt`

It is possible to filter on relations too, if `Offer` has a `Product` relation:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;

#[ApiResource]
#[ApiFilter(SearchFilter::class, properties: ['product' => 'exact'])]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
    offer.search_filter:
        parent: 'api_platform.doctrine.orm.search_filter'
        arguments: [ { product: 'exact' } ]
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined with inverted values.
        # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
        autowire: false
        autoconfigure: false
        public: false

# config/api/Offer.yaml
App\Entity\Offer:
    # ...
    collectionOperations:
        get:
            filters: ['offer.search_filter']
```

</code-selector>

With this service definition, it is possible to find all offers belonging to the product identified by a given IRI.
Try the following: `http://localhost:8000/api/offers?product=/api/products/12`.
Using a numeric ID is also supported: `http://localhost:8000/api/offers?product=12`

The above URLs will return all offers for the product having the following IRI as JSON-LD identifier (`@id`): `http://localhost:8000/api/products/12`.

### Date Filter

The date filter allows to filter a collection by date intervals.

Syntax: `?property[<after|before|strictly_after|strictly_before>]=value`

The value can take any date format supported by the [`\DateTime` constructor](https://www.php.net/manual/en/datetime.construct.php).

The `after` and `before` filters will filter including the value whereas `strictly_after` and `strictly_before` will filter excluding the value.

Like others filters, the date filter must be explicitly enabled:

<code-selector>
```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter;

#[ApiResource]
#[ApiFilter(DateFilter::class, properties: ['createdAt'])]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
    offer.date_filter:
        parent: 'api_platform.doctrine.orm.date_filter'
        arguments: [ { createdAt: ~ } ]
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined with inverted values.
        # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
        autowire: false
        autoconfigure: false
        public: false

# config/api/Offer.yaml
App\Entity\Offer:
    # ...
    collectionOperations:
        get:
            filters: ['offer.date_filter']
```

</code-selector>

Given that the collection endpoint is `/offers`, you can filter offers by date with the following query: `/offers?createdAt[after]=2018-03-19`.

It will return all offers where `createdAt` is superior or equal to `2018-03-19`.

#### Managing `null` Values

The date filter is able to deal with date properties having `null` values.
Four behaviors are available at the property level of the filter:

Description                          | Strategy to set
-------------------------------------|------------------------------------------------------------------------------------
Use the default behavior of the DBMS | `null`
Exclude items                        | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::EXCLUDE_NULL` (`exclude_null`)
Consider items as oldest             | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::INCLUDE_NULL_BEFORE` (`include_null_before`)
Consider items as youngest           | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::INCLUDE_NULL_AFTER` (`include_null_after`)
Always include items                 | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::INCLUDE_NULL_BEFORE_AND_AFTER` (`include_null_before_and_after`)

For instance, exclude entries with a property value of `null` with the following service definition:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter;

#[ApiResource]
#[ApiFilter(DateFilter::class, properties: ['dateProperty' => DateFilter::EXCLUDE_NULL])]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
    offer.date_filter:
        parent: 'api_platform.doctrine.orm.date_filter'
        arguments: [ { dateProperty: exclude_null } ]
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined with inverted values.
        # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
        autowire: false
        autoconfigure: false
        public: false

# config/api/Offer.yaml
App\Entity\Offer:
    # ...
    collectionOperations:
        get:
            filters: ['offer.date_filter']
```

</code-selector>

### Boolean Filter

The boolean filter allows you to search on boolean fields and values.

Syntax: `?property=<true|false|1|0>`

Enable the filter:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\BooleanFilter;

#[ApiResource]
#[ApiFilter(BooleanFilter::class, properties: ['isAvailableGenericallyInMyCountry'])]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
    offer.boolean_filter:
        parent: 'api_platform.doctrine.orm.boolean_filter'
        arguments: [ { isAvailableGenericallyInMyCountry: ~ } ]
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined with inverted values.
        # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
        autowire: false
        autoconfigure: false
        public: false

# config/api/Offer.yaml
App\Entity\Offer:
    # ...
    collectionOperations:
        get:
            filters: ['offer.boolean_filter']
```

</code-selector>

Given that the collection endpoint is `/offers`, you can filter offers with the following query: `/offers?isAvailableGenericallyInMyCountry=true`.

It will return all offers where `isAvailableGenericallyInMyCountry` equals `true`.

### Numeric Filter

The numeric filter allows you to search on numeric fields and values.

Syntax: `?property=<int|bigint|decimal...>`

Enable the filter:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\NumericFilter;

#[ApiResource]
#[ApiFilter(NumericFilter::class, properties: ['sold'])]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
    offer.numeric_filter:
        parent: 'api_platform.doctrine.orm.numeric_filter'
        arguments: [ { sold: ~ } ]
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined with inverted values.
        # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
        autowire: false
        autoconfigure: false
        public: false

# config/api/Offer.yaml
App\Entity\Offer:
    # ...
    collectionOperations:
        get:
            filters: ['offer.numeric_filter']
```

</code-selector>

Given that the collection endpoint is `/offers`, you can filter offers with the following query: `/offers?sold=1`.

It will return all offers with `sold` equals `1`.

### Range Filter

The range filter allows you to filter by a value lower than, greater than, lower than or equal, greater than or equal and between two values.

Syntax: `?property[<lt|gt|lte|gte|between>]=value`

Enable the filter:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\RangeFilter;

#[ApiResource]
#[ApiFilter(RangeFilter::class, properties: ['price'])]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
    offer.range_filter:
        parent: 'api_platform.doctrine.orm.range_filter'
        arguments: [ { price: ~ } ]
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined with inverted values.
        # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
        autowire: false
        autoconfigure: false
        public: false

# config/api/Offer.yaml
App\Entity\Offer:
    # ...
    collectionOperations:
        get:
            filters: ['offer.range_filter']
```

</code-selector>

Given that the collection endpoint is `/offers`, you can filter the price with the following query: `/offers?price[between]=12.99..15.99`.

It will return all offers with `price` between 12.99 and 15.99.

You can filter offers by joining two values, for example: `/offers?price[gt]=12.99&price[lt]=19.99`.

### Exists Filter

The exists filter allows you to select items based on a nullable field value.
It will also check the emptiness of a collection association.

Syntax: `?exists[property]=<true|false|1|0>`

Previous syntax (deprecated): `?property[exists]=<true|false|1|0>`

Enable the filter:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\ExistsFilter;

#[ApiResource]
#[ApiFilter(ExistsFilter::class, properties: ['transportFees'])]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
    offer.exists_filter:
        parent: 'api_platform.doctrine.orm.exists_filter'
        arguments: [ { transportFees: ~ } ]
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined with inverted values.
        # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
        autowire: false
        autoconfigure: false
        public: false

# config/api/Offer.yaml
App\Entity\Offer:
    # ...
    collectionOperations:
        get:
            filters: ['offer.exists_filter']
```

</code-selector>

Given that the collection endpoint is `/offers`, you can filter offers on nullable field with the following query: `/offers?exists[transportFees]=true`.

It will return all offers where `transportFees` is not `null`.

#### Using a Custom Exists Query Parameter Name

A conflict will occur if `exists` is also the name of a property with the search filter enabled.
Luckily, the query parameter name to use is configurable:

```yaml
# config/packages/api_platform.yaml
api_platform:
    collection:
        exists_parameter_name: 'not_null' # the URL query parameter to use is now "not_null"
```

### Order Filter (Sorting)

The order filter allows to sort a collection against the given properties.

Syntax: `?order[property]=<asc|desc>`

Enable the filter:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter;

#[ApiResource]
#[ApiFilter(OrderFilter::class, properties: ['id', 'name'], arguments: ['orderParameterName' => 'order'])]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
    offer.order_filter:
        parent: 'api_platform.doctrine.orm.order_filter'
        arguments:
            $properties: { id: ~, name: ~ }
            $orderParameterName: order
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined with inverted values.
        # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
        autowire: false
        autoconfigure: false
        public: false

# config/api/Offer.yaml
App\Entity\Offer:
    # ...
    collectionOperations:
        get:
            filters: ['offer.order_filter']
```

</code-selector>

Given that the collection endpoint is `/offers`, you can filter offers by name in ascending order and then by ID in descending
order with the following query: `/offers?order[name]=desc&order[id]=asc`.

By default, whenever the query does not specify the direction explicitly (e.g.: `/offers?order[name]&order[id]`), filters
will not be applied unless you configure a default order direction to use:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter;

#[ApiResource]
#[ApiFilter(OrderFilter::class, properties: ['id' => 'ASC', 'name' => 'DESC'])]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
    offer.order_filter:
        parent: 'api_platform.doctrine.orm.order_filter'
        arguments: [ { id: 'ASC', name: 'DESC' } ]
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined with inverted values.
        # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
        autowire: false
        autoconfigure: false
        public: false

# config/api/Offer.yaml
App\Entity\Offer:
    # ...
    collectionOperations:
        get:
            filters: ['offer.order_filter']
```

</code-selector>

#### Comparing with Null Values

When the property used for ordering can contain `null` values, you may want to specify how `null` values are treated in
the comparison:

Description                          | Strategy to set
-------------------------------------|---------------------------------------------------------------------------------------------
Use the default behavior of the DBMS | `null`
Consider items as smallest           | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter::NULLS_SMALLEST` (`nulls_smallest`)
Consider items as largest            | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter::NULLS_LARGEST` (`nulls_largest`)

For instance, treat entries with a property value of `null` as the smallest, with the following service definition:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter;

#[ApiResource]
#[ApiFilter(OrderFilter::class, properties: ['validFrom' => ['nulls_comparison' => OrderFilter::NULLS_SMALLEST, 'default_direction' => 'DESC']])]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
    offer.order_filter:
        parent: 'api_platform.doctrine.orm.order_filter'
        arguments: [ { validFrom: { nulls_comparison: 'nulls_smallest', default_direction: 'DESC' } } ]
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined with inverted values.
        # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
        autowire: false
        autoconfigure: false
        public: false

# config/api/Offer.yaml
App\Entity\Offer:
    # ...
    collectionOperations:
        get:
            filters: ['offer.order_filter']
```

</code-selector>

#### Using a Custom Order Query Parameter Name

A conflict will occur if `order` is also the name of a property with the search filter enabled.
Luckily, the query parameter name to use is configurable:

```yaml
# config/packages/api_platform.yaml
api_platform:
    collection:
        order_parameter_name: '_order' # the URL query parameter to use is now "_order"
```

### Filtering on Nested Properties

Sometimes, you need to be able to perform filtering based on some linked resources (on the other side of a relation). All
built-in filters support nested properties using the dot (`.`) syntax, e.g.:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;

#[ApiResource]
#[ApiFilter(OrderFilter::class, properties: ['product.releaseDate'])]
#[ApiFilter(SearchFilter::class, properties: ['product.color' => 'exact'])]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
    offer.order_filter:
        parent: 'api_platform.doctrine.orm.order_filter'
        arguments: [ { product.releaseDate: ~ } ]
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined with inverted values.
        # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
        autowire: false
        autoconfigure: false
        public: false
    offer.search_filter:
        parent: 'api_platform.doctrine.orm.search_filter'
        arguments: [ { product.color: 'exact' } ]
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined with inverted values.
        # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
        autowire: false
        autoconfigure: false
        public: false

# config/api/Offer.yaml
App\Entity\Offer:
    # ...
    collectionOperations:
        get:
            filters: ['offer.order_filter', 'offer.search_filter']
```

</code-selector>

The above allows you to find offers by their respective product's color: `http://localhost:8000/api/offers?product.color=red`,
or order offers by the product's release date: `http://localhost:8000/api/offers?order[product.releaseDate]=desc`

### Enabling a Filter for All Properties of a Resource

As we have seen in previous examples, properties where filters can be applied must be explicitly declared. If you don't
care about security and performance (e.g. an API with restricted access), it is also possible to enable built-in filters
for all properties:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter;

#[ApiResource]
#[ApiFilter(OrderFilter::class)]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
    offer.order_filter:
        parent: 'api_platform.doctrine.orm.order_filter'
        arguments: [ ~ ] # Pass null to enable the filter for all properties
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined with inverted values.
        # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
        autowire: false
        autoconfigure: false
        public: false

# config/api/Offer.yaml
App\Entity\Offer:
    # ...
    collectionOperations:
        get:
            filters: ['offer.order_filter']
```

</code-selector>

**Note: Filters on nested properties must still be enabled explicitly, in order to keep things sane.**

Regardless of this option, filters can be applied on a property only if:

* the property exists
* the value is supported (ex: `asc` or `desc` for the order filters).

It means that the filter will be **silently** ignored if the property:

* does not exist
* is not enabled
* has an invalid value

## Elasticsearch Filters

### Ordering Filter (Sorting)

The order filter allows to [sort](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-sort.html)
a collection against the given properties.

Syntax: `?order[property]=<asc|desc>`

Enable the filter:

<code-selector>

```php
<?php
// api/src/Model/Tweet.php

namespace App\Model;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\OrderFilter;

#[ApiResource]
#[ApiFilter(OrderFilter::class, properties: ['id', 'date'], arguments: ['orderParameterName' => 'order'])]
class Tweet
{
    // ...
}
```

```yaml
# config/services.yaml
services:
    tweet.order_filter:
        parent: 'api_platform.doctrine.orm.order_filter'
        arguments:
            $properties: { id: ~, date: ~ }
            $orderParameterName: 'order'
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined with inverted values.
        # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
        autowire: false
        autoconfigure: false
        public: false

# config/api/Tweet.yaml
App\Entity\Tweet:
    # ...
    filters: ['tweet.order_filter']
```

</code-selector>

Given that the collection endpoint is `/tweets`, you can filter tweets by id and date in ascending or descending order:
`/tweets?order[id]=asc&order[date]=desc`.

By default, whenever the query does not specify the direction explicitly (e.g: `/tweets?order[id]&order[date]`), filters
will not be applied unless you configure a default order direction to use:

```php
<?php
// api/src/Model/Tweet.php

namespace App\Model;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\OrderFilter;

#[ApiResource]
#[ApiFilter(OrderFilter::class, properties: ['id' => 'asc', 'date' => 'desc'])]
class Tweet
{
    // ...
}
```

#### Using a Custom Order Query Parameter Name (Elastic)

A conflict will occur if `order` is also the name of a property with the term filter enabled. Luckily, the query
parameter name to use is configurable:

```yaml
# config/packages/api_platform.yaml
api_platform:
    collection:
        order_parameter_name: '_order' # the URL query parameter to use is now "_order"
```

### Match Filter

The match filter allows to find resources that [match](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html)
the specified text on full text fields.

Syntax: `?property[]=value`

Enable the filter:

```php
<?php
// api/src/Model/Tweet.php

namespace App\Model;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\MatchFilter;

#[ApiResource]
#[ApiFilter(MatchFilter::class, properties: ['message'])]
class Tweet
{
    // ...
}
```

Given that the collection endpoint is `/tweets`, you can filter tweets by message content.

`/tweets?message=Hello%20World` will return all tweets that match the text `Hello World`.

### Term Filter

The term filter allows to find resources that contain the exact specified
[terms](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-term-query.html).

Syntax: `?property[]=value`

Enable the filter:

```php
<?php
// api/src/Model/User.php

namespace App\Model;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\TermFilter;

#[ApiResource]
#[ApiFilter(TermFilter::class, properties: ['gender', 'age'])]
class User
{
    // ...
}
```

Given that the collection endpoint is `/users`, you can filter users by gender and age.

`/users?gender=female` will return all users whose gender is `female`.
`/users?age=42` will return all users whose age is `42`.

Filters can be combined together: `/users?gender=female&age=42`.

### Filtering on Nested Properties (Elastic)

Sometimes, you need to be able to perform filtering based on some linked resources (on the other side of a relation).
All built-in filters support nested properties using the (`.`) syntax.

```php
<?php
// api/src/Model/Tweet.php

namespace App\Model;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\OrderFilter;
use ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\TermFilter;

#[ApiResource]
#[ApiFilter(OrderFilter::class, properties: ['author.firstName'])]
#[ApiFilter(TermFilter::class, properties: ['author.gender'])]
class Tweet
{
    // ...
}
```

The above allows you to find tweets by their respective author's gender `/tweets?author.gender=male`, or order tweets by the
author's first name `/tweets?order[author.firstName]=desc`.

## Serializer Filters

### Group Filter

The group filter allows you to filter by serialization groups.

Syntax: `?groups[]=<group>`

You can add as many groups as you need.

Enable the filter:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Serializer\Filter\GroupFilter;

#[ApiResource]
#[ApiFilter(GroupFilter::class, arguments: ['parameterName' => 'groups', 'overrideDefaultGroups' => false, 'whitelist' => ['allowed_group']])]
class Book
{
    // ...
}
```

Three arguments are available to configure the filter:

* `parameterName` is the query parameter name (default `groups`)
* `overrideDefaultGroups` allows to override the default serialization groups (default `false`)
* `whitelist` groups whitelist to avoid uncontrolled data exposure (default `null` to allow all groups)

Given that the collection endpoint is `/books`, you can filter by serialization groups with the following query: `/books?groups[]=read&groups[]=write`.

### Property filter

**Note:** We strongly recommend using [Vulcain](https://vulcain.rocks) instead of this filter.
Vulcain is faster, allows a better hit rate, and is supported out of the box in the API Platform distribution.

The property filter adds the possibility to select the properties to serialize (sparse fieldsets).

Syntax: `?properties[]=<property>&properties[<relation>][]=<property>`

You can add as many properties as you need.

Enable the filter:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Serializer\Filter\PropertyFilter;

#[ApiResource]
#[ApiFilter(PropertyFilter::class, arguments: ['parameterName' => 'properties', 'overrideDefaultProperties' => false, 'whitelist' => ['allowed_property']])]
class Book
{
    // ...
}
```

Three arguments are available to configure the filter:

* `parameterName` is the query parameter name (default `properties`)
* `overrideDefaultProperties` allows to override the default serialization properties (default `false`)
* `whitelist` properties whitelist to avoid uncontrolled data exposure (default `null` to allow all properties)

Given that the collection endpoint is `/books`, you can filter the serialization properties with the following query: `/books?properties[]=title&properties[]=author`.
If you want to include some properties of the nested "author" document, use: `/books?properties[]=title&properties[author][]=name`.

## Creating Custom Filters

Custom filters can be written by implementing the `ApiPlatform\Core\Api\FilterInterface` interface.

API Platform provides a convenient way to create Doctrine ORM and MongoDB ODM filters. If you use [custom data providers](data-providers.md),
you can still create filters by implementing the previously mentioned interface, but - as API Platform isn't aware of your
persistence system's internals - you have to create the filtering logic by yourself.

### Creating Custom Doctrine ORM Filters

Doctrine ORM filters have access to the context created from the HTTP request and to the `QueryBuilder` instance used to
retrieve data from the database. They are only applied to collections. If you want to deal with the DQL query generated
to retrieve items, [extensions](extensions.md) are the way to go.

A Doctrine ORM filter is basically a class implementing the `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\FilterInterface`.
API Platform includes a convenient abstract class implementing this interface and providing utility methods: `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\AbstractFilter`.

In the following example, we create a class to filter a collection by applying a regexp to a property. The `REGEXP` DQL
function used in this example can be found in the [`DoctrineExtensions`](https://github.com/beberlei/DoctrineExtensions)
library. This library must be properly installed and registered to use this example (works only with MySQL).

```php
<?php
// api/src/Filter/RegexpFilter.php

namespace App\Filter;

use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\AbstractContextAwareFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use Doctrine\ORM\QueryBuilder;
use Symfony\Component\PropertyInfo\Type;

final class RegexpFilter extends AbstractContextAwareFilter
{
    protected function filterProperty(string $property, $value, QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, string $operationName = null)
    {
        // otherwise filter is applied to order and page as well
        if (
            !$this->isPropertyEnabled($property, $resourceClass) ||
            !$this->isPropertyMapped($property, $resourceClass)
        ) {
            return;
        }

        $parameterName = $queryNameGenerator->generateParameterName($property); // Generate a unique parameter name to avoid collisions with other filters
        $queryBuilder
            ->andWhere(sprintf('REGEXP(o.%s, :%s) = 1', $property, $parameterName))
            ->setParameter($parameterName, $value);
    }

    // This function is only used to hook in documentation generators (supported by Swagger and Hydra)
    public function getDescription(string $resourceClass): array
    {
        if (!$this->properties) {
            return [];
        }

        $description = [];
        foreach ($this->properties as $property => $strategy) {
            $description["regexp_$property"] = [
                'property' => $property,
                'type' => Type::BUILTIN_TYPE_STRING,
                'required' => false,
                'description' => 'Filter using a regex. This will appear in the OpenApi documentation!',
                'openapi' => [
                    'example' => 'Custom example that will be in the documentation and be the default value of the sandbox',
                    'allowReserved' => false,// if true, query parameters will be not percent-encoded
                    'allowEmptyValue' => true,
                    'explode' => false, // to be true, the type must be Type::BUILTIN_TYPE_ARRAY, ?product=blue,green will be ?product=blue&product=green
                ],
            ];
        }

        return $description;
    }
}
```

Thanks to [Symfony's automatic service loading](https://symfony.com/doc/current/service_container.html#service-container-services-load-example), which is enabled by default in the API Platform distribution, the filter is automatically registered as a service!

Finally, add this filter to resources you want to be filtered by using the `ApiFilter` attribute:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use App\Filter\RegexpFilter;

#[ApiResource]
#[ApiFilter(RegexpFilter::class)]
class Offer
{
    // ...
}
```

You can now use this filter in the URL like `http://example.com/offers?regexp_email=^[FOO]`. This new filter will also
appear in OpenAPI and Hydra documentations.

In the previous example, the filter can be applied on any property. You can also apply this filter on a specific property:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use App\Filter\RegexpFilter;

#[ApiResource]
class Offer
{
    // ...

    #[ApiFilter(RegexpFilter::class)]
    public string $name;
}
```

#### Manual Service and Attribute Registration

If you don't use Symfony's automatic service loading, you have to register the filter as a service by yourself.
Use the following service definition (remember, by default, this isn't needed!):

```yaml
# config/services.yaml
services:
    # ...
    # This whole definition can be omitted if automatic service loading is enabled
    'App\Filter\RegexpFilter':
        # The "arguments" key can be omitted if the autowiring is enabled
        arguments: [ '@doctrine', ~, '@?logger' ]
        # The "tags" key can be omitted if the autoconfiguration is enabled
        tags: [ 'api_platform.filter' ]
```

In the previous example, the filter can be applied on any property. However, thanks to the `AbstractFilter` class,
it can also be enabled for some properties:

```yaml
# config/services.yaml
services:
    'App\Filter\RegexpFilter':
        arguments: [ '@doctrine', ~, '@?logger', { email: ~, anOtherProperty: ~ } ]
        tags: [ 'api_platform.filter' ]
```

Finally, if you don't want to use the `#[ApiFilter]` attribute, you can register the filter on an API resource class using the `filters` attribute:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use App\Filter\RegexpFilter;

#[ApiResource(
    attributes => [
        'filters' => [RegexpFilter::class],
    ],
)]
class Offer
{
    // ...
}
```

### Creating Custom Doctrine MongoDB ODM Filters

Doctrine MongoDB ODM filters have access to the context created from the HTTP request and to the [aggregation builder](https://www.doctrine-project.org/projects/doctrine-mongodb-odm/en/latest/reference/aggregation-builder.html)
instance used to retrieve data from the database and to execute [complex operations on data](https://docs.mongodb.com/manual/aggregation/).
They are only applied to collections. If you want to deal with the aggregation pipeline generated to retrieve items, [extensions](extensions.md) are the way to go.

A Doctrine MongoDB ODM filter is basically a class implementing the `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Filter\FilterInterface`.
API Platform includes a convenient abstract class implementing this interface and providing utility methods: `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Filter\AbstractFilter`.

### Creating Custom Elasticsearch Filters

Elasticsearch filters have access to the context created from the HTTP request and to the Elasticsearch query clause.
They are only applied to collections. If you want to deal with the query DSL through the search request body, extensions
are the way to go.

Existing Elasticsearch filters are applied through a [constant score query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-constant-score-query.html).
A constant score query filter is basically a class implementing the `ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\ConstantScoreFilterInterface`
and the `ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\FilterInterface`. API Platform includes a convenient
abstract class implementing this last interface and providing utility methods: `ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\AbstractFilter`.

Suppose you want to use the [match filter](#match-filter) on a property named `$fullName` and you want to add the [and operator](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html#query-dsl-match-query-boolean) to your query:

```php
<?php
// api/src/ElasticSearch/AndOperatorFilterExtension.php

namespace App\ElasticSearch;

use ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Extension\RequestBodySearchCollectionExtensionInterface;

class AndOperatorFilterExtension implements RequestBodySearchCollectionExtensionInterface
{
    public function applyToCollection(array $requestBody, string $resourceClass, ?string $operationName = null, array $context = []): array
    {
        $requestBody['query'] = $requestBody['query'] ?? [];
        $andQuery = [
            'query' => $context['filters']['fullName'],
            'operator' => 'and',
        ];
        
        $requestBody['query']['constant_score']['filter']['bool']['must'][0]['match']['full_name'] = $andQuery;
        
        return $requestBody;
    }
}
```

### Using Doctrine ORM Filters

Doctrine ORM features [a filter system](http://docs.doctrine-project.org/projects/doctrine-orm/en/latest/reference/filters.html) that allows the developer to add SQL to the conditional clauses of queries, regardless of the place where the SQL is generated (e.g. from a DQL query, or by loading associated entities).
These are applied on collections and items and therefore are incredibly useful.

The following information, specific to Doctrine filters in Symfony, is based upon [a great article posted on Michaël Perrin's blog](http://blog.michaelperrin.fr/2014/12/05/doctrine-filters/).

Suppose we have a `User` entity and an `Order` entity related to the `User` one. A user should only see his orders and no one else's.

```php
<?php
// api/src/Entity/User.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource]
class User
{
    // ...
}
```

```php
<?php
// api/src/Entity/Order.php

namespace App\Entity;

use App\Entity\User;
use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

#[ApiResource]
class Order
{
    // ...

    #[ORM\ManyToOne(User::class)]
    #[ORM\JoinColumn(name: "user_id", referencedColumnName: "id")]
    public User $user;
    
    // ...
}
```

The whole idea is that any query on the order table should add a `WHERE user_id = :user_id` condition.

Start by creating a custom attribute to mark restricted entities:

```php
<?php
// api/Annotation/UserAware.php

namespace App\Attribute;

use Attribute;

#[Attribute(Attribute::TARGET_CLASS)]
final class UserAware
{
    public $userFieldName;
}
```

Then, let's mark the `Order` entity as a "user aware" entity.

```php
<?php
// api/src/Entity/Order.php

namespace App\Entity;

use App\Attribute\UserAware;

#[UserAware(userFieldName: "user_id")]
class Order {
    // ...
}
```

Now, create a Doctrine filter class:

```php
<?php
// api/src/Filter/UserFilter.php

namespace App\Filter;

use App\Attribute\UserAware;
use Doctrine\ORM\Mapping\ClassMetadata;
use Doctrine\ORM\Query\Filter\SQLFilter;

final class UserFilter extends SQLFilter
{
    public function addFilterConstraint(ClassMetadata $targetEntity, $targetTableAlias): string
    {
        // The Doctrine filter is called for any query on any entity
        // Check if the current entity is "user aware" (marked with an attribute)
        $userAware = $targetEntity->getReflectionClass()->getAttributes(UserAware::class)[0] ?? null;

        $fieldName = $userAware?->getArguments()['userFieldName'] ?? null;
        if ($fieldName === '' || is_null($fieldName)) {
            return '';
        }

        try {
            // Don't worry, getParameter automatically escapes parameters
            $userId = $this->getParameter('id');
        } catch (\InvalidArgumentException $e) {
            // No user id has been defined
            return '';
        }

        if (empty($fieldName) || empty($userId)) {
            return '';
        }

        return sprintf('%s.%s = %s', $targetTableAlias, $fieldName, $userId);
    }
}
```

Now, we must configure the Doctrine filter.

```yaml
# config/packages/api_platform.yaml
doctrine:
    orm:
        filters:
            user_filter:
                class: App\Filter\UserFilter
                enabled: true
```

Done: Doctrine will automatically filter all `UserAware`entities!

## ApiFilter Attribute

The attribute can be used on a `property` or on a `class`.

If the attribute is given over a property, the filter will be configured on the property. For example, let's add a search filter on `name` and on the `prop` property of the `colors` relation:

```php
<?php

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;
use App\Entity\DummyCarColor;

#[ApiResource]
class DummyCar
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    #[ORM\Column]
    #[ApiFilter(SearchFilter::class, strategy: 'partial')]
    public ?string $name = null;

    #[ORM\OneToMany(mappedBy: "car", targetEntity: DummyCarColor::class)]
    #[ApiFilter(SearchFilter::class, properties: ['colors.prop' => 'ipartial'])]
    public Collection $colors;

    public function __construct()
    {
        $this->colors = new ArrayCollection();
    }

    // ...
}
```

On the first property, `name`, it's straightforward. The first attribute argument is the filter class, the second specifies options, here, the strategy:

```php
#[ApiFilter(SearchFilter::class, strategy: 'partial')]
```

In the second attribute, we specify `properties` on which the filter should apply. It's necessary here because we don't want to filter `colors` but the `prop` property of the `colors` association.
Note that for each given property we specify the strategy:

```php
#[ApiFilter(SearchFilter::class, properties: ['colors.prop' => 'ipartial'])]
```

The `ApiFilter` attribute can be set on the class as well. If you don't specify any properties, it'll act on every property of the class.

For example, let's define three data filters (`DateFilter`, `SearchFilter` and `BooleanFilter`) and two serialization filters (`PropertyFilter` and `GroupFilter`) on our `DummyCar` class:

```php
<?php
// api/src/Entity/DummyCar.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\BooleanFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;
use ApiPlatform\Core\Serializer\Filter\GroupFilter;
use ApiPlatform\Core\Serializer\Filter\PropertyFilter;
use Doctrine\ORM\Mapping as ORM;

#[ApiResource]
#[ApiFilter(BooleanFilter::class)]
#[ApiFilter(DateFilter::class, strategy: DateFilter::EXCLUDE_NULL)]
#[ApiFilter(SearchFilter::class, properties: ['colors.prop' => 'ipartial', 'name' => 'partial'])]
#[ApiFilter(PropertyFilter::class, arguments: ['parameterName' => 'foobar'])]
#[ApiFilter(GroupFilter::class, arguments: ['parameterName' => 'foobargroups'])]
class DummyCar
{
    // ...
}

```

The `BooleanFilter` is applied to every `Boolean` property of the class. Indeed, in each core filter we check the Doctrine type. It's written only by using the filter class:

```php
#[ApiFilter(BooleanFilter::class)]
```

The `DateFilter` given here will be applied to every `Date` property of the `DummyCar` class with the `DateFilter::EXCLUDE_NULL` strategy:

```php
#[ApiFilter(DateFilter::class, strategy: DateFilter::EXCLUDE_NULL)]
```

The `SearchFilter` here adds properties. The result is the exact same as the example with attributes on properties:

```php
#[ApiFilter(SearchFilter::class, properties: ['colors.prop' => 'ipartial', 'name' => 'partial'])]
```

Note that you can specify the `properties` argument on every filter.

The next filters are not related to how the data is fetched but rather to how the serialization is done on those. We can give an `arguments` option ([see here for the available arguments](#serializer-filters)):

```php
#[ApiFilter(PropertyFilter::class, arguments: ['parameterName' => 'foobar'])]
#[ApiFilter(GroupFilter::class, arguments: ['parameterName' => 'foobargroups'])]
```
