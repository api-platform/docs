# URL Generation Strategy

By default, API Platform generates all URLs as absolute paths to the base URL.

For instance, in JSON-LD, you will get a collection like this:

```json
{
  "@context": "/contexts/Book",
  "@id": "/books",
  "@type": "Collection",
  "member": [
    {
      "@id": "/books/1",
      "@type": "https://schema.org/Book",
      "name": "My awesome book"
    }
  ],
  "totalItems": 1
}
```

You may want to use absolute URLs (for instance if resources are used in another API) or network paths instead.

## Configure URL Generation Globally

It can be configured globally using one of the configurations below:

### Configure URL Generation Globally using Symfony

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  defaults:
    url_generation_strategy: !php/const ApiPlatform\Api\UrlGeneratorInterface::ABS_URL
```
### Configure URL Generation Globally using Laravel

```php
<?php
// config/api-platform.php
return [
    // ....
    'defaults' => [
        'url_generation_strategy' => ApiPlatform\Api\UrlGeneratorInterface::ABS_URL
    ],
];
```

## Configure URL Generation for a Specific Resource

It can also be configured only for a specific resource:

<code-selector>

```php
<?php
// api/src/ApiResource/Book.php with Symfony or app/ApiResource/Book.php with Laravel

namespace App\ApiResource;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Api\UrlGeneratorInterface;

#[ApiResource(urlGenerationStrategy: UrlGeneratorInterface::ABS_URL)]
class Book
{
    // ...
}
```

```yaml
# api/config/api_platform/resources.yaml
# The YAML syntax is only supported for Symfony
resources:
  App\ApiResource\Book:
    urlGenerationStrategy: !php/const ApiPlatform\Api\UrlGeneratorInterface::ABS_URL
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->
<!-- The XML syntax is only supported for Symfony -->

<resources
        xmlns="https://api-platform.com/schema/metadata/resources-3.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
        https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\ApiResource\Book" urlGenerationStrategy="0" />
</resources>
```

</code-selector>

For the above configuration, the collection will be like this:

```json
{
  "@context": "http://example.com/contexts/Book",
  "@id": "http://example.com/books",
  "@type": "Collection",
  "member": [
    {
      "@id": "http://example.com/books/1",
      "@type": "https://schema.org/Book",
      "name": "My awesome book"
    }
  ],
  "totalItems": 1
}
```
