# JSON Schema Support

[JSON Schema](https://json-schema.org/) is a popular vocabulary to describe the shape of JSON documents. A variant of JSON Schema is also used [in OpenAPI specifications](swagger.md).

API Platform provides an infrastructure to generate JSON Schemas for any resource, represented in any format (including JSON-LD).
The generated schema can be used with libraries such as [react-json-schema-form](https://github.com/rjsf-team/react-jsonschema-form) to build forms for the documented resources, or to [be used for validation](https://json-schema.org/implementations.html#validators).

## Generating a JSON Schema

To export the schema corresponding to an API Resource, run the following command:

```console
docker-compose exec php \
    bin/console api:json-schema:generate 'App\Entity\Book'
```

To see all options available, try:

```console
docker-compose exec php \
    bin/console help api:json-schema:generate
```

## Specifications concerning unit testing functions based on the JSON Schema

In a unit testing context, API Platform does not use the same schema version than the schema used when generating the API documentation. The version used by the documentation is the OpenAPI Schema version and the version used by unit testing is the JSON Schema version.

When [Testing the API](testing.md), JSON Schemas are usefull to generate and automate unit testing. API Platform provide specific unit testing functionnalities like [`assertMatchesResourceCollectionJsonSchema`](testing.md#writing-functional-tests) or [`assertMatchesResourceItemJsonSchema`](testing.md#writing-functional-tests) functions. These functions generate a JSON Schema then do unit testing based on the generated schema automatically.

Usually the fact that API Platform use a different schema version for unit testing is not a problem, but sometime you may need to use the [`ApiProperty`](openapi.md#using-the-openapi-and-swagger-contexts) attribute to specify a [calculated field](serialization.md#calculated-field) type by overriding the OpenAPI Schema for the calculated field to be correctly documented. But when you will use [`assertMatchesResourceCollectionJsonSchema`](testing.md#writing-functional-tests) or [`assertMatchesResourceItemJsonSchema`](testing.md#writing-functional-tests) functions the unit test will fail on this [calculated field](serialization.md#calculated-field) as the unit test process doesn't use the `openapi_context` you specified because API Platform is using the JSON Schema version instead at this moment.

So there is a way to override JSON Schema specification for a specific property in the JSON Schema used by the unit testing process.

You will need to add the `json_schema_context` property in the [`ApiProperty`](openapi.md#using-the-openapi-and-swagger-contexts) attribute do this, example :

```php
<?php

declare(strict_types=1);

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ORM\Entity
 */
#[ApiResource(
    collectionOperations: [
        'get' => ['normalization_context' => ['groups' => 'greeting:collection:get']],
    ],
)]
class Greeting
{
    /**
     * @var int The entity Id
     *
     * @ORM\Id
     * @ORM\GeneratedValue
     * @ORM\Column(type="integer")
     */
    #[Groups("greeting:collection:get")]
    private $id;
    
    private $a = 1;
    
    private $b = 2;

    /**
     * @var string A nice person
     *
     * @ORM\Column
     */
    #[Groups("greeting:collection:get")]
    public $name = '';

    public function getId(): int
    {
        return $this->id;
    }

    #[Groups("greeting:collection:get")]
    #[ApiProperty(attributes: [
        "openapi_context" => [
            "type" => "array",
            "items" => ["type" => "integer"]
        ],
        "json_schema_context" => [ // <- MAGIC IS HERE, you can override the json_schema_context here.
            "type" => "array",
            "items" => ["type" => "integer"]
        ]
    ])]
    public function getSomeNumbers(): array {
        return [1, 2, 3, 4];
    }
}
```



## Generating a JSON Schema Programmatically

To generate JSON Schemas programmatically, use the `api_platform.json_schema.schema_factory` [service](https://symfony.com/doc/current/service_container.html#fetching-and-using-services).

## Testing

API Platform provides a PHPUnit assertion to test if a response is valid according to a given Schema: `assertMatchesJsonSchema()`.
Refer to [the testing documentation](testing.md) for more details.
