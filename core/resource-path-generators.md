# Resource path generators

With API Platform, you can configure the path of your exposed resources. 
It comes with pre-registered generators, which you can easily override.

## Configuration

There are two pre-registered path generator services:

| strategy                                                  | entity name  | path result                       |
|-----------------------------------------------------------|--------------|-----------------------------------|
| `api_platform.routing.resource_path_generator.underscore` | `MyResource` | `/my_resources`                   |
| `api_platform.routing.resource_path_generator.dash`       | `MyResource` | `/my-resources`                   |

The default generator is `api_platform.routing.resource_path_generator.underscore`
To change it to dash generator for example, add the following lines to `app/config/config.yml`:

```yaml
api_platform:
    routing:
        resource_path_generator: api_platform.routing.resource_path_generator.dash
```


## Create a custom path generator

Let's assumes we need urls without separators (e.g. `api.tld/myresources`)

### Defining the Resource Path Generator

Make sure the custom generator implements `ApiPlatform\Core\Routing\ResourcePathGeneratorInterface` : 

```php
<?php

// src/AppBundle/Routing/NoSeparatorsResourcePathGenerator.php

namespace AppBundle\Routing;

use ApiPlatform\Core\Routing\ResourcePathGeneratorInterface;
use Doctrine\Common\Inflector\Inflector;

class NoSeparatorsResourcePathGenerator implements ResourcePathGeneratorInterface
{
    public function generateResourceBasePath(string $resourceShortName) : string
    {
        $pathName = strtolower($resourceShortName);

        return Inflector::pluralize($pathName);
    }
}
```

Note that `$resourceShortName` contains a camel case string, by default the resource class name (e.g. `MyResource`).

### Registering the service

<configurations>

```yaml
# src/AppBundle/Resources/config/services.yml

services:
  app.routing.resource_path_generator.no_separators:
    class: 'AppBundle\Routing\NoSeparatorsResourcePathGenerator'
```

```xml
<!-- src/AppBundle/Resources/config/services.xml -->

<?xml version="1.0" encoding="UTF-8" ?>
<services>
    <service id="app.routing.resource_path_generator.no_separators" class="AppBundle\Routing\NoSeparatorsResourcePathGenerator" /> 
</services>
```

</configurations>

### Configure it

```yaml
# app/config/config.yml

api_platform:
    routing:
        resource_path_generator: app.routing.resource_path_generator.no_separators
```

Previous chapter: [Serialization groups and relations](serialization-groups-and-relations.md)
Next chapter: [Validation](validation.md)