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

## Client-Generated IDs

The [JSON:API specification allows clients to supply their own `id` on `POST`](https://jsonapi.org/format/#crud-creating-client-ids)
when the server agrees to accept it. This is useful when the client generates a UUID before
sending the request and needs the server to persist it as-is.

API Platform refuses client-supplied `data.id` on `POST` by default. Accepting arbitrary IDs on a
public endpoint without opt-in is a footgun: a malicious client could overwrite existing records or
enumerate internal identifiers. Sending `data.id` on a `POST` without opt-in results in a `400`
response.

### Enabling Globally

**Symfony:**

```yaml
# config/packages/api_platform.yaml
api_platform:
    jsonapi:
        allow_client_generated_id: true
```

**Laravel** (`config/api-platform.php`):

```php
'jsonapi' => [
    'allow_client_generated_id' => true,
],
```

### Enabling Per Operation

Use the `denormalizationContext` on the `#[Post]` operation to enable the feature for a single
endpoint without affecting the rest of the API:

```php
<?php

namespace App\ApiResource;

use ApiPlatform\JsonApi\Serializer\ItemNormalizer;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\Post;

#[ApiResource(
    formats: ['jsonapi' => ['application/vnd.api+json']],
    operations: [
        new Get(uriTemplate: '/books/{id}'),
        new Post(
            uriTemplate: '/books',
            denormalizationContext: [ItemNormalizer::ALLOW_CLIENT_GENERATED_ID => true],
        ),
    ],
)]
class Book
{
    public ?string $id = null;
    public string $title = '';
}
```

A request that supplies `data.id` is then accepted:

```http
POST /api/books HTTP/1.1
Accept: application/vnd.api+json
Content-Type: application/vnd.api+json

{
    "data": {
        "type": "Book",
        "id": "01932b4c-a3f1-7b7e-9e5b-3d8f1c2e4a6d",
        "attributes": {
            "title": "Hyperion"
        }
    }
}
```

The supplied `id` is passed to the entity's `id` setter. The processor is responsible for
persisting it. The response output schema still requires `id`; only the `POST` input schema
marks it as optional.
