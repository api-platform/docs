# JSON Schema Support

[JSON Schema](https://json-schema.org/) is a popular vocabulary to describe the shape of JSON documents. A variant of JSON Schema is also used
[in OpenAPI specifications](openapi.md).

API Platform provides an infrastructure to generate JSON Schemas for any resource, represented in any format
(including JSON-LD).
The generated schema can be used with libraries such as [react-json-schema-form](https://github.com/rjsf-team/react-jsonschema-form) to build forms for the documented
resources, or to [be used for validation](https://json-schema.org/implementations.html#validators).

## Generating a JSON Schema

> [!WARNING]
> These commands are not yet available with Laravel, you're welcome to contribute [on GitHub](https://github.com/api-platform/core)

To export the schema corresponding to an API Resource, run the following command:

```console
bin/console api:json-schema:generate 'App\ApiResource\Book'
```

To see all options available, try:

```console
bin/console help api:json-schema:generate
```

## Overriding the JSON Schema Specification

In a unit testing context, API Platform does not use the same schema version as the schema used when generating the API
documentation. The version used by the documentation is the OpenAPI Schema version and the version used by unit testing
is the JSON Schema version.

> [!NOTE]
> For assertions about JSON schemas in  Laravel, refer to the
> [API Test Assertions in Laravel documentation](../laravel/testing.md#api-test-assertions-with-laravel).

When [Testing the API](../core/testing.md), JSON Schemas are useful to generate and automate unit testing. API Platform provides specific
unit testing functionalities like [`assertMatchesResourceCollectionJsonSchema()`](../symfony/testing.md#writing-functional-tests) or
[`assertMatchesResourceItemJsonSchema()`](../symfony/testing.md#writing-functional-tests)  methods.
These methods generate a JSON Schema then do unit testing based on the generated schema automatically.

Usually, the fact that API Platform uses a different schema version for unit testing is not a problem, but sometimes you
may need to use the [`ApiProperty`](openapi.md#using-the-openapi-and-swagger-contexts) attribute to specify a [calculated field](serialization.md#calculated-field) type by overriding the OpenAPI Schema
for the calculated field to be correctly documented.

When you will use [`assertMatchesResourceCollectionJsonSchema()`](../symfony/testing.md#writing-functional-tests) or
[`assertMatchesResourceItemJsonSchema()`](../symfony/testing.md#writing-functional-tests) functions the unit test will
fail on this [calculated field](serialization.md#calculated-field) as the unit testing process doesn't use the `openapi_context`you specified because
API Platform is using the JSON Schema version instead at this moment.

So there is a way to override JSON Schema specification for a specific property in the JSON Schema used by the unit testing process.

You will need to add the `json_schema_context` property in the [`ApiProperty`](openapi.md#using-the-openapi-and-swagger-contexts)
attribute to do this, example:

```php
<?php

namespace App\ApiResource;

use ApiPlatform\Metadata\ApiResource;

#[ApiResource]
class Greeting
{
    #[ApiProperty(
        identifier: true,
        openapiContext: [
            'type' => 'integer',
            'example' => 1
        ]
    )]
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

To generate JSON Schemas programmatically, use the `api_platform.json_schema.schema_factory`.

For further information, please consult the following documentations:

- [Symfony: Fetching and Using Services](https://symfony.com/doc/current/service_container.html#fetching-and-using-services)
- [Laravel: Resolving Services](https://laravel.com/docs/container#resolving)  

## Testing

API Platform provides a PHPUnit assertion to test if a response is valid according to a given Schema: `assertMatchesJsonSchema()`.
Refer to [the testing documentation](../core/testing.md) for more details.
