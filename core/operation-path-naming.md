# Operation Path Naming

With API Platform Core, you can configure the default resolver used to generate operation paths.
Pre-registered resolvers are available and can easily be overridden.

## Configuration

There are two pre-registered operation path naming services:

| Service name                                                   | Entity name  | Path result     |
|----------------------------------------------------------------|--------------|-----------------|
| `api_platform.metadata.path_segment_name_generator.underscore` | `MyResource` | `/my_resources` |
| `api_platform.metadata.path_segment_name_generator.dash`       | `MyResource` | `/my-resources` |

The default resolver is `api_platform.metadata.path_segment_name_generator.underscore`.

### Configuration using Symfony

To change it to the dash resolver, add the following lines to `api/config/packages/api_platform.yaml`:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  path_segment_name_generator: api_platform.metadata.path_segment_name_generator.dash
```

### Configuration using Laravel

To change it to the dash resolver, add the following lines to `config/api-platform.php`:

```php
<?php
// config/api-platform.php
return [
    // ....
    'path_segment_name_generator' => 'api_platform.metadata.path_segment_name_generator.dash'
];
```

## Create a Custom Operation Path Resolver

Let's assume we need URLs without separators (e.g. `api.tld/myresources`)

### Defining the Operation Segment Name Generator

Make sure the custom segment generator implements [`ApiPlatform\Metadata\Operation\PathSegmentNameGeneratorInterface`](https://github.com/api-platform/core/blob/main/src/Metadata/Operation/PathSegmentNameGeneratorInterface.php):

```php
<?php
// api/src/Operation/SingularPathSegmentNameGenerator.php with Symfony or app/Operation/SingularPathSegmentNameGenerator.php with Laravel
namespace App\Operation;

use ApiPlatform\Metadata\Operation\PathSegmentNameGeneratorInterface;

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

Note that `$name` contains a camelCase string, by default the resource class name (e.g. `MyResource`).

### Registering the Service (for Symfony only)

If you haven't disabled the autowiring option, the service will be registered automatically and you have nothing more to
do.
Otherwise, you must register this class as a service like in the following example:

```yaml
# api/config/services.yaml
services:
  # ...
  'App\Operation\SingularPathSegmentNameGenerator': ~
```

### Configuring the Service

#### Configuring It using Symfony

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  path_segment_name_generator: 'App\Operation\SingularPathSegmentNameGenerator'
```

#### Configuring It using Laravel

```php
<?php
// config/api-platform.php
return [
    // ....
    'path_segment_name_generator' => App\Operation\SingularPathSegmentNameGenerator::class
];
```
