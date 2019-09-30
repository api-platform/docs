# JSON Schema Support

[JSON Schema](https://json-schema.org/) is a popular vocabulary to describe the shape of JSON documents. A variant of JSON Schema is also used [in OpenAPI specifications](swagger.md).

API Platform provides an infrastructure to generate JSON Schemas for any resource, represented in any format (including JSON-LD). 
The generated schema can be used with libraries such as [react-json-schema-form](https://github.com/rjsf-team/react-jsonschema-form) to build forms for the documented resources, or to [be used for validation](https://json-schema.org/implementations.html#validators).

## Generating a JSON Schema

To export the schema corresponding to an API Resource, run the following command:

    $ docker-compose exec php bin/console api:json-schema:generate 'App\Entity\Book'

To see all options available, try:

    $ docker-compose exec php bin/console help api:json-schema:generate

## Generating a JSON Schema Programmatically

To generate JSON Schemas programmatically, use the `api_platform.json_schema.schema_factory` [service](https://symfony.com/doc/current/service_container.html#fetching-and-using-services).

## Testing

API Platform provides a PHPUnit assertion to test if a response is valid according to a given Schema: `assertMatchesJsonSchema()`.
Refer to [the testing documentation](testing.md) for more details.
