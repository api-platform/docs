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
To change it to the dash resolver, add the following lines to `config/packages/api_platform.yaml`:

```yaml
# config/packages/api_platform.yaml
api_platform:
    path_segment_name_generator: api_platform.path_segment_name_generator.dash
```

## Create a Custom Operation Path Resolver

Let's assume we need URLs without separators (e.g. `api.tld/myresources`)

### Defining the Operation Segment Name Generator

Make sure the custom segment generator implements [`ApiPlatform\Core\Operation\PathSegmentNameGeneratorInterface`](https://github.com/api-platform/core/blob/main/src/Operation/PathSegmentNameGeneratorInterface.php):

```php
<?php

// api/src/Operation/SingularPathSegmentNameGenerator.php

namespace App\Operation;

use ApiPlatform\Core\Operation\PathSegmentNameGeneratorInterface;

class SingularPathSegmentNameGenerator implements PathSegmentNameGeneratorInterface
{
    /**
     * Transforms a given string to a valid path name which can be pluralized (eg. for collections).
     *
     * @param string $name usually a ResourceMetadata shortname
     *
     * @return string A string that is a part of the route name
     */
    public function getSegmentName(string $name, bool $collection = true): string
    {
        $name = $this->dashize($name);

        return $name;
    }

    private function dashize(string $string): string
    {
        return strtolower(preg_replace('~(?<=\\w)([A-Z])~', '-$1', $string));
    }
}
```

Note that `$name` contains a camel case string, by default the resource class name (e.g. `MyResource`).

### Registering the Service

If you haven't disabled the autowiring option, the service will be registered automatically and you have nothing more to
do.
Otherwise, you must register this class as a service like in the following example:

```yaml
# config/services.yaml
services:
    # ...
    'App\Operation\SingularPathSegmentNameGenerator': ~
```

### Configuring It

```yaml
# config/packages/api_platform.yaml
api_platform:
    path_segment_name_generator: 'App\Operation\SingularPathSegmentNameGenerator'
```
