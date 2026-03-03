# JSON:API

API Platform supports the [JSON:API](https://jsonapi.org) format. When a client sends a request with
an `Accept: application/vnd.api+json` header, API Platform serializes responses following the
JSON:API specification.

For details on enabling formats, see [content negotiation](content-negotiation.md).

## Entity Identifiers as Resource IDs

We recommend configuring API Platform to use entity identifiers as the `id` field of JSON:API
resource objects. This will become the default in 5.x:

```yaml
# config/packages/api_platform.yaml
api_platform:
    jsonapi:
        use_iri_as_id: false
```

With this configuration, the JSON:API `id` field contains the entity identifier (e.g., `"10"`)
instead of the full IRI (e.g., `"/dummies/10"`). A `links.self` field is added to each resource
object for navigation:

```json
{
    "data": {
        "id": "10",
        "type": "Dummy",
        "links": {
            "self": "/dummies/10"
        },
        "attributes": {
            "name": "Dummy #10"
        },
        "relationships": {
            "relatedDummy": {
                "data": {
                    "id": "1",
                    "type": "RelatedDummy"
                }
            }
        }
    }
}
```

Relationships reference related resources by entity identifier and `type`.

### Composite Identifiers

Resources with composite identifiers use a semicolon-separated string as the `id` value (e.g.,
`"field1=val1;field2=val2"`).

### Resources Without a Standalone Item Endpoint

API Platform must resolve the IRI for any resource that appears in a relationship. If a resource has
no standalone `GET` item endpoint (for example, it is only exposed as a subresource), IRI resolution
fails.

Use the `NotExposed` operation to register a URI template for internal IRI resolution without
exposing a public endpoint. A `NotExposed` operation registers the route internally but returns a
`404` response when accessed directly:

```php
<?php

namespace App\Entity;

use ApiPlatform\Metadata\NotExposed;

#[NotExposed(uriTemplate: '/tags/{id}')]
class Tag
{
    public int $id;
    public string $label;
}
```

This allows a parent resource to reference `Tag` objects in its relationships while `Tag` itself has
no public item endpoint. The `NotExposed` operation requires a `uriTemplate` with a single URI
variable.
