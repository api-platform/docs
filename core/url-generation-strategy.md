# URL Generation Strategy

By default, API Platform generates all URLs as absolute paths to the base URL.

For instance, in JSON-LD, you will get a collection like this:

```json
{
  "@context": "/contexts/Book",
  "@id": "/books",
  "@type": "hydra:Collection",
  "hydra:member": [
    {
      "@id": "/books/1",
      "@type": "https://schema.org/Book",
      "name": "My awesome book"
    }
  ],
  "hydra:totalItems": 1
}
```

You may want to use absolute URLs (for instance if resources are used in another API) or network paths instead.

It can be configured globally:

```yaml
# config/packages/api_platform.yaml
api_platform:
    defaults:
        url_generation_strategy: !php/const ApiPlatform\Core\Api\UrlGeneratorInterface::ABS_URL
```

It can also be configured only for a specific resource:

<code-selector>

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Api\UrlGeneratorInterface;

#[ApiResource(urlGenerationStrategy: UrlGeneratorInterface::ABS_URL)]
class Book
{
    // ...
}
```

```yaml
# config/api_platform/resources.yaml
App\Entity\Book:
    attributes:
        url_generation_strategy: !php/const ApiPlatform\Core\Api\UrlGeneratorInterface::ABS_URL
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources
        xmlns="https://api-platform.com/schema/metadata"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata
        https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="App\Entity\Book">
        <attribute name="url_generation_strategy" type="constant">ApiPlatform\Core\Api\UrlGeneratorInterface::ABS_URL</attribute>
    </resource>
</resources>
```

</code-selector>

For the above configuration, the collection will be like this:

```json
{
  "@context": "http://example.com/contexts/Book",
  "@id": "http://example.com/books",
  "@type": "hydra:Collection",
  "hydra:member": [
    {
      "@id": "http://example.com/books/1",
      "@type": "https://schema.org/Book",
      "name": "My awesome book"
    }
  ],
  "hydra:totalItems": 1
}
```
