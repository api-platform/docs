# Client Integrations

## Edge Side API (ESA)

> [Edge Side APIs (ESA)](https://edge-side-api.rocks/) is an architectural pattern that allows the creation of more
> reliable, efficient, and less resource-intensive APIs. It revives the core REST/HATEOAS principles while taking full
> advantage of the new capabilities provided by the web platform.
>
> ESA promotes a mixed approach (synchronous and asynchronous), offering simplicity in development and use, exceptional
> performance, and the ability for clients to receive real-time updates of the resources they fetched. ESA also leverages
> existing standards to expose API documentation, enabling the creation of generic clients capable of discovering the
> API’s capabilities at runtime.
>
> — *From [ESA White Paper](https://edge-side-api.rocks/white-paper)*

## JavaScript Client Integrations

API Platform offers a suite of tools and libraries that streamline the integration of JavaScript clients with APIs.
These tools simplify development by automating tasks such as data fetching, administration panel creation,
and real-time updates. Below is a detailed overview of the available clients, libraries, and their usage.

### Clients and Tools Overview

#### Admin

API Platform Admin is a dynamic administration panel generator built with [React-Admin](https://marmelab.com/react-admin/).
It automatically adapts to your API schema and provides extensive customization options. It can read an [OpenAPI](https://www.openapis.org/)
specification or a [Hydra](https://www.hydra-cg.com/) specification. API Platform supports both [OpenAPI](openapi.md) and
[Hydra](extending-jsonld-context.md#hydra) from scratch!

[Learn more about API Platform Admin](../admin/index.md).

#### Create Client

The Client Generator creates JavaScript/TypeScript clients based on your API documentation. It generates code that
integrates seamlessly with your API endpoints, reducing development time and errors.

[Learn more about the Create Client](../create-client/index.md)

### JavaScript Libraries

#### api-platform/ld

The [api-platform/ld](https://edge-side-api.rocks/linked-data) JavaScript library simplifies working with Linked Data.
It helps parse and serialize data in formats such as [JSON-LD](extending-jsonld-context.md#json-ld), making it easier to
handle complex relationships in your applications.

For example, let's load authors when required with a Linked Data approach.
Given an API referencing books and their authors, where `GET /books/1` returns:

```json
{
    "@id": "/books/1",
    "@type": ["https://schema.org/Book"],
    "title": "Hyperion",
    "author": "https://localhost/authors/1"
}
```

Use an [URLPattern](https://urlpattern.spec.whatwg.org/) to load authors automatically when fetching an author property
such as `books.author?.name`:

```javascript
import ld from '@api-platform/ld'

const pattern = new URLPattern("/authors/:id", "https://localhost");
const books = await ld('/books', {
    urlPattern: pattern,
    onUpdate: (newBooks) => {
        log()
    }
})

function log() {
    console.log(books.author?.name)
}

log()
```

With [api-platform/ld](https://edge-side-api.rocks/linked-data), authors are automatically loaded when needed.

[Read the full documentation](https://edge-side-api.rocks/linked-data).

#### api-platform/mercure

[Mercure](https://mercure.rocks/spec) is a real-time communication protocol. The [api-platform/mercure](https://edge-side-api.rocks/mercure)
library enables you to subscribe to updates and deliver real-time data seamlessly.

Our frontend library allows you to subscribe to updates with efficiency, re-using the hub connection and adding topics
automatically as they get requested. API Platform [supports Mercure](mercure.md) and automatically sets the
[Link header](https://mercure.rocks/spec#content-negotiation) making auto-discovery a breeze. For example:

```javascript
import mercure, { close } from "@api-platform/mercure";

const res = await mercure('https://localhost/authors/1', {
    onUpdate: (author) => console.log(author)
})

const author = res.then(res => res.json())

// Close if you need to 
history.onpushstate = function(e) {
    close('https://localhost/authors/1')
}
```

Assuming `/authors/1` returned the following:

```http
(docs(client-integration): introduce documentation)
162eae78c8a (fix(markdown): change code block type to http)
Link: <https://localhost/authors/1>; rel="self"
Link: <https://localhost/.well-known/mercure>; rel="mercure"
```

An `EventSource` subscribes to the topic `https://localhost/authors/1` on the hub `https://localhost/.well-known/mercure`.

[Read the full documentation](https://edge-side-api.rocks/mercure).

#### api-platform/api-doc-parser

The [api-platform/api-doc-parser](https://github.com/api-platform/api-doc-parser)  that parses Hydra, Swagger,
OpenAPI, and GraphQL documentation into an intermediate format for generating API clients and scaffolding code.
It integrates well with API Platform and supports auto-detecting resource relationships.

Key Features:

- Multi-format support: Parses Hydra, Swagger (OpenAPI v2), OpenAPI v3, and GraphQL.
- Intermediate representation: Converts API docs into a usable format for generating clients, scaffolding code, or building admin interfaces.
- API Platform integration: Works seamlessly with API Platform.
- Auto-detection of resource relationships: Automatically detects relationships between resources based on documentation.

Example: Parsing [Hydra](http://hydra-cg.com/) API Documentation:

```javascript
import { parseHydraDocumentation } from '@api-platform/api-doc-parser';

parseHydraDocumentation('https://demo.api-platform.com').then(({api}) => console.log(api));
```
This example fetches Hydra documentation from `https://demo.api-platform.com`, parses it, and logs the resulting API
structure. The `parseHydraDocumentation` method is particularly useful for building metadata-driven clients or handling advanced API interactions.

[Read the full documentation](https://github.com/api-platform/api-doc-parser).
