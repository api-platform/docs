# OpenAPI

API Platform Admin has native support for API exposing an [OpenAPI documentation](https://www.openapis.org/).

To use it, use the `OpenApiAdmin` component, with the entry point of the API and the entry point of the OpenAPI documentation in JSON:

```javascript
import { OpenApiAdmin } from '@api-platform/admin';

export default () => (
  <OpenApiAdmin
    entrypoint="https://demo.api-platform.com"
    docEntrypoint="https://demo.api-platform.com/docs.jsonopenapi"
  />
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
import { OpenApiAdmin } from '@api-platform/admin';
import drfProvider from 'ra-data-django-rest-framework';

export default () => (
  <OpenApiAdmin
    dataProvider={drfProvider('https://django-api.com')}
    entrypoint="https://django-api.com"
    docEntrypoint="https://django-api.com/docs.json"
  />
);
```

### Custom Data Provider

For more advanced use cases, you can create a custom dataProvider to add features such as authentication,
logging, or token-based authorization.

Here's an example of how to integrate a simpleRestProvider with a customized httpClient:

```javascript
import { fetchUtils } from 'react-admin';
import { openApiDataProvider } from '@api-platform/admin';
import simpleRestProvider from 'ra-data-simple-rest';
import { getAccessToken } from './accessToken';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost/api';
const DOC_URL = `${API_URL}/docs`;

// Custom HTTP client for authenticated requests
const httpClient = async (url, options = {}) => {
  options.headers = new Headers({
    ...options.headers,
    Accept: 'application/json',
  });

  const token = getAccessToken();
  if (token) {
    options.user = { token: `Bearer ${token}`, authenticated: true };
  }

  try {
    const { status, headers, body, json } = await fetchUtils.fetchJson(url, options);
    return { status, headers, body, json };
  } catch (error) {
    throw error;
  }
};

// Wrapping the custom HTTP client into a data provider
const jsonDataProvider = openApiDataProvider({
  dataProvider: simpleRestProvider(API_URL, httpClient),
  entrypoint: API_URL,
  docEntrypoint: DOC_URL,
});

export default () => (
  <OpenApiAdmin
    entrypoint={API_URL}
    docEntrypoint={DOC_URL}
    dataProvider={jsonDataProvider}
  />
);

```

> [!NOTE]
> The `getAccessToken` function is a placeholder for your custom logic to retrieve a JWT token.
>
> Implement this function according to your application's requirements, such as reading the token from local storage,
> cookies, or a secure context.

## Mercure Support

Mercure support can be enabled manually by giving the `mercure` prop to the `OpenApiAdmin` component.

See also [the dedicated section](real-time-mercure.md).
