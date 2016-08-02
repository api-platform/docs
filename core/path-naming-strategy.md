# Path Naming Strategy

With API Platform Core, you can configure the naming strategy of URL paths exposing your resources globally.
It comes with pre-registered generators, which you can easily override.

## Configuration

There are two pre-registered path generator services:

Strategy                                                       | Entity name  | Path result
---------------------------------------------------------------|--------------|----------------
`api_platform.naming.resource_path_naming_strategy.underscore` | `MyResource` | `/my_resources`
`api_platform.naming.resource_path_naming_strategy.dash`       | `MyResource` | `/my-resources`

The default generator is `api_platform.naming.resource_path_naming_strategy.underscore`.
To change it to the dash generator, add the following lines to `app/config/config.yml`:

```yaml
# app/config/config.yml

api_platform:
    naming:
        resource_path_naming_strategy: 'api_platform.naming.resource_path_naming_strategy.dash'

```

## Create a custom path generator

Let's assumes we need urls without separators (e.g. `api.tld/myresources`)

### Defining the Resource Path Generator

Make sure the custom generator implements `[ApiPlatform\Core\Naming\ResourcePathNamingStrategyInterface](https://github.com/api-platform/core/blob/master/src/Naming/ResourcePathNamingStrategyInterface.php)`:

```php
// src/AppBundle/Naming/NoSeparatorsResourcePathGenerator.php

namespace AppBundle\Naming;

use ApiPlatform\Core\Naming\ResourcePathNamingStrategyInterface;
use Doctrine\Common\Inflector\Inflector;

final class NoSeparatorsResourcePathGenerator implements ResourcePathNamingStrategyInterface
{
    public function generateResourceBasePath(string $resourceShortName) : string
    {
        return Inflector::pluralize(strtolower($resourceShortName));
    }
}
```

Note that `$resourceShortName` contains a camel case string, by default the resource class name (e.g. `MyResource`).

### Registering the service

<configurations>

```yaml
# src/AppBundle/Resources/config/services.yml

services:
  app.naming.resource_path_naming_strategy.no_separators:
    class: 'AppBundle\Naming\NoSeparatorsResourcePathGenerator'
    public: false
```

```xml
<!-- src/AppBundle/Resources/config/services.xml -->

<?xml version="1.0" encoding="UTF-8" ?>
<services>
    <service id="app.naming.resource_path_naming_strategy.no_separators" class="AppBundle\Naming\NoSeparatorsResourcePathGenerator" public="false" />
</services>
```

</configurations>

### Configure it

```yaml
# app/config/config.yml

api_platform:
    naming:
        resource_path_naming_strategy: 'app.naming.resource_path_naming_strategy.no_separators'

```

Previous chapter: [Serialization groups and relations](serialization-groups-and-relations.md)
Next chapter: [Validation](validation.md)
