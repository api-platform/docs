# OpenAPI

API Platform Admin has native support for API exposing an [OpenAPI documentation](https://www.openapis.org/).

To use it, use the `OpenApiAdmin` component, with the entry point of the API and the entry point of the OpenAPI documentation in JSON:

```javascript
import { OpenApiAdmin } from "@api-platform/admin";

export default () => (
  <OpenApiAdmin entrypoint="https://demo.api-platform.com" docEntrypoint="https://demo.api-platform.com/docs.jsonopenapi" />
);
```

> [!NOTE]
>
> The OpenAPI documentation needs to follow some assumptions to be understood correctly by the underlying `api-doc-parser`.
> See the [dedicated part in the `api-doc-parser` library README](https://github.com/api-platform/api-doc-parser#openapi-support).

## Data Provider

By default, the component will use a basic data provider, without pagination support.

If you want to use [another data provider](https://marmelab.com/react-admin/DataProviderList.html), pass the `dataProvider` prop to the component:

```javascript
import { OpenApiAdmin } from "@api-platform/admin";
import drfProvider from "ra-data-django-rest-framework";

export default () => (
  <OpenApiAdmin
    dataProvider={drfProvider("https://django-api.com")}
    entrypoint="https://django-api.com"
    docEntrypoint="https://django-api.com/docs.json"
  />
);
```

## Mercure Support

Mercure support can be enabled manually by giving the `mercure` prop to the `OpenApiAdmin` component.

See also [the dedicated section](real-time-mercure.md).
