# Operation Path Naming

With API Platform Core, you can configure the default resolver used to generate operation paths.
Pre-registered resolvers are available and can easily be overridden.

## Configuration

There are two pre-registered operation path naming services:

Service name                                      | Entity name  | Path result
--------------------------------------------------|--------------|----------------
`api_platform.operation_path_resolver.underscore` | `MyResource` | `/my_resources`
`api_platform.operation_path_resolver.dash`       | `MyResource` | `/my-resources`

The default resolver is `api_platform.operation_path_resolver.underscore`.
To change it to the dash resolver, add the following lines to `app/config/config.yml`:

```yaml
# app/config/config.yml

api_platform:
    default_operation_path_resolver: 'api_platform.operation_path_resolver.dash'
```

## Create a Custom Operation Path Resolver

Let's assume we need URLs without separators (e.g. `api.tld/myresources`)

### Defining the Operation Path Resolver

Make sure the custom resolver implements [`ApiPlatform\Core\PathResolver\OperationPathResolverInterface`](https://github.com/api-platform/core/blob/master/src/PathResolver/OperationPathResolverInterface.php):

```php
<?php

// src/AppBundle/PathResolver/NoSeparatorsOperationPathResolver.php

namespace AppBundle\PathResolver;

use ApiPlatform\Core\PathResolver\OperationPathResolverInterface;
use Doctrine\Common\Inflector\Inflector;

final class NoSeparatorsOperationPathResolver implements OperationPathResolverInterface
{
    public function resolveOperationPath(string $resourceShortName, array $operation, bool $collection) : string
    {
        $path = Inflector::pluralize(strtolower($resourceShortName));
        if (!$collection) {
            $path .= '/{id}';
        }
        $path .= '.{_format}';

        return $path;
    }
}
```

Note that `$resourceShortName` contains a camel case string, by default the resource class name (e.g. `MyResource`).

### Registering the Service

<configurations>

```yaml
# app/config/services.yml

services:
    app.operation_path_resolver.no_separators:
        class: 'AppBundle\\PathResolver\\NoSeparatorsOperationPathResolver'
        public: false
```

```xml
<!-- app/config/services.xml -->

<?xml version="1.0" encoding="UTF-8" ?>
<services>
    <service id="app.operation_path_resolver.no_separators" class="AppBundle\PathResolver\NoSeparatorsOperationPathResolver" public="false" />
</services>
```

</configurations>

### Configure It

```yaml
# app/config/config.yml

api_platform:
    default_operation_path_resolver: 'app.operation_path_resolver.no_separators'
```

Previous chapter: [Performance](performance.md)

Next chapter: [Accept application/x-www-form-urlencoded Form Data](form-data.md)
