# Swagger / Open API Support

API Platform natively support the [Open API](https://www.openapis.org/) (formerly Swagger) API documentation format.
It also integrates a customized version of [Swagger UI](https://swagger.io/swagger-ui/), a nice tool to display the
API documentation in a user friendly way.

![Screenshot](../distribution/images/swagger-ui-1.png)

## Overriding the Swagger Documentation

Symfony allows to [decorate services](https://symfony.com/doc/current/service_container/service_decoration.html), here we
need to decorate `api_platform.swagger.normalizer.documentation`.

In the following example, we will see how to override the title of the Swagger documentation and add a custom filter for
the `GET` operation of `/foos` path

```yaml
# app/config/services.yml
services:
    'AppBundle\Swagger\SwaggerDecorator':
        decorates: 'api_platform.swagger.normalizer.documentation'
        arguments: [ '@AppBundle\Swagger\SwaggerDecorator.inner' ]
        autoconfigure: false
```

```php
<?php
// src/AppBundle/Swagger/SwaggerDecorator.php

namespace AppBundle\Swagger;

use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

final class SwaggerDecorator implements NormalizerInterface
{
    private $decorated;

    public function __construct(NormalizerInterface $decorated)
    {
        $this->decorated = $decorated;
    }

    public function normalize($object, $format = null, array $context = [])
    {
        $docs = $this->decorated->normalize($object, $format, $context);

        $customDefinition = [
            'name' => 'fields',
            'definition' => 'Fields to remove of the outpout',
            'default' => 'id',
            'in' => 'query',
        ];


	// e.g. add a custom parameter
		$docs['paths']['/foos']['get']['parameters'][] = $customDefinition;

		// Override title
		$docs['info']['title'] = 'My Api Foo';

        return $docs;
    }

    public function supportsNormalization($data, $format = null)
    {
        return $this->decorated->supportsNormalization($data, $format);
    }
}
```

## Using the Swagger Context

Sometimes you may want to have additional information included in your Swagger documentation.
The following configuration will provide additional context to your Swagger definitions:

```php
<?php
// src/AppBundle/Entity/Product.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Annotation\ApiProperty;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * @ApiResource
 * @ORM\Entity
 */
class Product // The class name will be used to name exposed resources
{
    /**
     * @ORM\Column(type="integer")
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    public $id;

    /**
     * @param string $name A name property - this description will be avaliable in the API documentation too.
     *
     * @ORM\Column
     * @Assert\NotBlank
     *
     * @ApiProperty(
     *     "attributes"={
     *         "swagger_context"={
     *             "type"="string",
     *             "enum"={"one", "two"},
     *             "example"="one"          
     *         }
     *     }
     * )
     */
    public $name;

    /**
     * @ORM\Column
     * @Assert\DateTime
     *
     * @ApiProperty(
     *     "attributes"={
     *         "swagger_context"={"type"="string", "format"="date-time"}
     *     }
     * )
     */
    public $timestamp;
}
```

Or in YAML:

```yaml
# src/AppBundle/Resources/config/api_resources/resources.yml
resources:
    AppBundle\Entity\Product:
      properties:
        name:
          attributes:
            swagger_context:
              type: string
              enum: ['one', 'two']
              example: one
        timestamp:
          attributes:
            swagger_context:
              type: string
              format: date-time
```

Will produce the following Swagger documentation:
```json
{
  "swagger": "2.0",
  "basePath": "/",

  "definitions": {
    "Product": {
      "type": "object",
      "description": "This is a product.",
      "properties": {
        "id": {
          "type": "integer",
          "readOnly": true
        },
        "name": {
          "type": "string",
          "description": "This is a name.",
          "enum": ["one", "two"],
          "example": "one"
        },
        "timestamp": {
          "type": "string",
          "format": "date-time"
        }
      }
    }
  }
}
```
