# Operation Path Naming

With API Platform Core, you can configure the default resolver used to generate operation paths.
Pre-registered resolvers are available and can easily be overridden.

## Configuration

There are two pre-registered operation path naming services:

Service name                                          | Entity name  | Path result
------------------------------------------------------|--------------|----------------
`api_platform.path_segment_name_generator.underscore` | `MyResource` | `/my_resources`
`api_platform.path_segment_name_generator.dash`       | `MyResource` | `/my-resources`

The default resolver is `api_platform.path_segment_name_generator.underscore`.
To change it to the dash resolver, add the following lines to `api/config/packages/api_platform.yaml`:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    path_segment_name_generator: api_platform.path_segment_name_generator.dash
```

## Create a Custom Operation Path Resolver

Let's assume we need URLs without separators (e.g. `api.tld/myresources`)

### Defining the Operation Path Resolver

Make sure the custom resolver implements [`ApiPlatform\Core\PathResolver\OperationPathResolverInterface`](https://github.com/api-platform/core/blob/master/src/PathResolver/OperationPathResolverInterface.php):

```php
<?php

// api/src/PathResolver/NoSeparatorsOperationPathResolver.php

namespace App\PathResolver;

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

If you haven't disabled the autowiring option, the service will be registered automatically and you have nothing more to
do.
Otherwise, you must register this class a service like in the following example:

```yaml
# api/config/services.yaml
services:
    # ...
    'App\PathResolver\NoSeparatorsOperationPathResolver': ~
```

### Configure It

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    path_segment_name_generator: 'App\PathResolver\NoSeparatorsOperationPathResolver'
```
