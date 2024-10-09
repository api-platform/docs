# JSON Schema Support

[JSON Schema](https://json-schema.org/) is a popular vocabulary to describe the shape of JSON documents. A variant of JSON Schema is also used [in OpenAPI specifications](openapi.md).

API Platform provides an infrastructure to generate JSON Schemas for any resource, represented in any format (including JSON-LD).
The generated schema can be used with libraries such as [react-json-schema-form](https://github.com/rjsf-team/react-jsonschema-form) to build forms for the documented resources, or to [be used for validation](https://json-schema.org/implementations.html#validators).

## Generating a JSON Schema

To export the schema corresponding to an API Resource, run the following command:

```console
docker compose exec php \
    bin/console api:json-schema:generate 'App\Entity\Book'
```

To see all options available, try:

```console
docker compose exec php \
    bin/console help api:json-schema:generate
```

## Overriding the JSON Schema Specification

In a unit testing context, API Platform does not use the same schema version as the schema used when generating the API documentation. The version used by the documentation is the OpenAPI Schema version and the version used by unit testing is the JSON Schema version.

When [Testing the API](testing.md), JSON Schemas are useful to generate and automate unit testing. API Platform provides specific unit testing functionalities like [`assertMatchesResourceCollectionJsonSchema()`](testing.md#writing-functional-tests) or [`assertMatchesResourceItemJsonSchema()`](testing.md#writing-functional-tests) methods.
These methods generate a JSON Schema then do unit testing based on the generated schema automatically.

Usually, the fact that API Platform uses a different schema version for unit testing is not a problem, but sometimes you may need to use the [`ApiProperty`](openapi.md#using-the-openapi-and-swagger-contexts) attribute to specify a [calculated field](serialization.md#calculated-field) type by overriding the OpenAPI Schema for the calculated field to be correctly documented.

When you will use [`assertMatchesResourceCollectionJsonSchema()`](testing.md#writing-functional-tests) or [`assertMatchesResourceItemJsonSchema()`](testing.md#writing-functional-tests) functions the unit test will fail on this [calculated field](serialization.md#calculated-field) as the unit testing process doesn't use the `openapi_context` you specified
because API Platform is using the JSON Schema version instead at this moment.

So there is a way to override JSON Schema specification for a specific property in the JSON Schema used by the unit testing process.

You will need to add the `json_schema_context` property in the [`ApiProperty`](openapi.md#using-the-openapi-and-swagger-contexts) attribute to do this, example:

```php
<?php

namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource]
class Greeting
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    // [...]

    public function getId(): ?int
    {
        return $this->id;
    }

    #[ApiProperty(
        openapiContext: [
            'type' => 'array',
            'items' => ['type' => 'integer']
        ],
        jsonSchemaContext: [
            'type' => 'array',
            'items' => ['type' => 'integer']
        ]
    )]
    public function getSomeNumbers(): array {
        return [1, 2, 3, 4];
    }
}
```

You can obtain more information about the available [JSON Schema Types and format here](https://json-schema.org/understanding-json-schema/reference/type.html).

## Generating a JSON Schema Programmatically

To generate JSON Schemas programmatically, use the `api_platform.json_schema.schema_factory` [service](https://symfony.com/doc/current/service_container.html#fetching-and-using-services).

## Testing

API Platform provides a PHPUnit assertion to test if a response is valid according to a given Schema: `assertMatchesJsonSchema()`.
Refer to [the testing documentation](testing.md) for more details.
