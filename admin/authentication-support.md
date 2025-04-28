# Authentication Support

API Platform Admin delegates the authentication support to React Admin.

Refer to the [Auth Provider Setup](https://marmelab.com/react-admin/Authentication.html) documentation for more information.

**Tip:** Once you have set up the authentication, you can also configure React Admin to perform client-side Authorization checks. Refer to the [Authorization](https://marmelab.com/react-admin/Permissions.html) documentation for more information.

## HydraAdmin

Enabling authentication support for [`<HydraAdmin>` component](./components.md#hydra) consists of a few parts, which need to be integrated together.

In the following steps, we will see how to:

- Make authenticated requests to the API (i.e. include the `Authorization` header)
- Redirect users to the login page if they are not authenticated
- Clear expired tokens when encountering unauthorized `401` response

### Make Authenticated Requests

First, we need to implement a `getHeaders` function, that will add the Bearer token from `localStorage` (if there is one) to the `Authorization` header.

```typescript
const getHeaders = () =>
  localStorage.getItem("token")
    ? { Authorization: `Bearer ${localStorage.getItem("token")}` }
    : {};
```

Then, extend the Hydra `fetch` function to use the `getHeaders` function to add the `Authorization` header to the requests.

```typescript
import {
    fetchHydra as baseFetchHydra,
} from "@api-platform/admin";

const fetchHydra = (url, options = {}) =>
  baseFetchHydra(url, {
    ...options,
    headers: getHeaders,
  });

```

### Redirect To Login Page

Then, we'll create a `<RedirectToLogin>` component, that will redirect users to the `/login` route if no token is available in the `localStorage`, and call the dataProvider's `introspect` function otherwise.

```tsx
import { Navigate } from "react-router-dom";
import { useIntrospection } from "@api-platform/admin";

const RedirectToLogin = () => {
  const introspect = useIntrospection();

  if (localStorage.getItem("token")) {
    introspect();
    return <></>;
  }
  return <Navigate to="/login" />;
};
```

### Clear Expired Tokens

Now, we will extend the `parseHydraDocumentaion` function (imported from the [@api-platform/api-doc-parser](https://github.com/api-platform/api-doc-parser) library).

We will customize it to clear expired tokens when encountering unauthorized `401` response.

```typescript
import { parseHydraDocumentation } from "@api-platform/api-doc-parser";
import { ENTRYPOINT } from "config/entrypoint";

const apiDocumentationParser = (setRedirectToLogin) => async () => {
  try {
    setRedirectToLogin(false);
    return await parseHydraDocumentation(ENTRYPOINT, { headers: getHeaders });
  } catch (result) {
    const { api, response, status } = result;
    if (status !== 401 || !response) {
      throw result;
    }

    localStorage.removeItem("token");
    setRedirectToLogin(true);

    return { api, response, status };
  }
};
```

### Extend The Data Provider

Now, we can initialize the Hydra data provider with the custom `fetchHydra` (with custom headers) and `apiDocumentationParser` functions created earlier.

```typescript
import {
    hydraDataProvider as baseHydraDataProvider,
} from "@api-platform/admin";
import { ENTRYPOINT } from "config/entrypoint";

const dataProvider = (setRedirectToLogin) =>
  baseHydraDataProvider({
    entrypoint: ENTRYPOINT,
    httpClient: fetchHydra,
    apiDocumentationParser: apiDocumentationParser(setRedirectToLogin),
  });
```

### Update The Admin Component

Lastly, we can stitch everything together in the `Admin` component.

```tsx
// src/Admin.tsx

import Head from "next/head";
import { useState } from "react";
import { Navigate, Route } from "react-router-dom";
import { CustomRoutes } from "react-admin";
import {
    fetchHydra as baseFetchHydra,
    HydraAdmin,
    hydraDataProvider as baseHydraDataProvider,
    useIntrospection,
} from "@api-platform/admin";
import { parseHydraDocumentation } from "@api-platform/api-doc-parser";
import authProvider from "utils/authProvider";
import { ENTRYPOINT } from "config/entrypoint";

// Functions and components created in the previous steps:
const getHeaders = () => {...};
const fetchHydra = (url, options = {}) => {...};
const RedirectToLogin = () => {...};
const apiDocumentationParser = (setRedirectToLogin) => async () => {...};
const dataProvider = (setRedirectToLogin) => {...};

export const Admin = () => {
  const [redirectToLogin, setRedirectToLogin] = useState(false);

  return (
    <>
      <Head>
        <title>API Platform Admin</title>
      </Head>

      <HydraAdmin
        dataProvider={dataProvider(setRedirectToLogin)}
        authProvider={authProvider}
        entrypoint={window.origin}
      >
        {redirectToLogin ? (
          <CustomRoutes>
            <Route path="/" element={<RedirectToLogin />} />
            <Route path="/:any" element={<RedirectToLogin />} />
          </CustomRoutes>
        ) : (
          <>
            <Resource name=".." list="..">
            <Resource name=".." list="..">
          </>
        )}
      </HydraAdmin>
    </>
  );
};
```

### Example Implementation

For the implementation of the admin component, you can find a working example in the [API Platform's demo application](https://github.com/api-platform/demo/blob/4.0/pwa/components/admin/Admin.tsx).

## OpenApiAdmin

This section explains how to set up and customize the [`<OpenApiAdmin>` component](./components.md/#openapi) to enable authentication.

In the following steps, we will see how to:

- Make authenticated requests to the API (i.e. include the `Authorization` header)
- Implement an authProvider to redirect users to the login page if they are not authenticated, and clear expired tokens when encountering unauthorized `401` response

### Making Authenticated Requests

First, we need to create a custom `httpClient` to add authentication tokens (via the the `Authorization` HTTP header) to requests.

We will then configure `openApiDataProvider` to use [`ra-data-simple-rest`](https://github.com/marmelab/react-admin/blob/master/packages/ra-data-simple-rest/README.md), a simple REST dataProvider for React Admin, and make it use the `httpClient` we created earlier.

```typescript
// src/dataProvider.ts

const getAccessToken = () => localStorage.getItem("token");

const httpClient = async (url: string, options: fetchUtils.Options = {}) => {
    options.headers = new Headers({
        ...options.headers,
        Accept: 'application/json',
    }) as Headers;

    const token = getAccessToken();
    options.user = { token: `Bearer ${token}`, authenticated: !!token };

    return await fetchUtils.fetchJson(url, options);
};

const dataProvider = openApiDataProvider({
  dataProvider: simpleRestProvider(API_ENTRYPOINT_PATH, httpClient),
  entrypoint: API_ENTRYPOINT_PATH,
  docEntrypoint: API_DOCS_PATH,
});
```

**Note:** The `simpleRestProvider` provider expect the API to include a `Content-Range` header in the response. You can find more about the header syntax in the [Mozilla’s MDN documentation: Content-Range](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Range).

**Note:** The `getAccessToken` function retrieves the JWT token stored in the browser's localStorage. Replace it with your own logic in case you don't store the token that way.

### Creating The AuthProvider

Now let's create and export an `authProvider` object that handles authentication and authorization logic.

```typescript
// src/authProvider.ts

interface JwtPayload {
    sub: string;
    username: string;
}

const getAccessToken = () => localStorage.getItem("token");

const authProvider = {
    login: async ({username, password}: { username: string; password: string }) => {
        const request = new Request(API_AUTH_PATH, {
            method: "POST",
            body: JSON.stringify({ email: username, password }),
            headers: new Headers({ "Content-Type": "application/json" }),
        });

        const response = await fetch(request);

        if (response.status < 200 || response.status >= 300) {
            throw new Error(response.statusText);
        }

        const auth = await response.json();
        localStorage.setItem("token", auth.token);
    },
    logout: () => {
        localStorage.removeItem("token");
        return Promise.resolve();
    },
    checkAuth: () => getAccessToken() ? Promise.resolve() : Promise.reject(),
    checkError: (error: { status: number }) => {
        const status = error.status;
        if (status === 401 || status === 403) {
            localStorage.removeItem("token");
            return Promise.reject();
        }

        return Promise.resolve();
    },
    getIdentity: () => {
        const token = getAccessToken();

        if (!token) return Promise.reject();

        const decoded = jwtDecode<JwtPayload>(token);

        return Promise.resolve({
            id: decoded.sub,
            fullName: decoded.username,
            avatar: "",
        });
    },
    getPermissions: () => Promise.resolve(""),
};

export default authProvider;
```

### Updating The Admin Component

Finally, we can update the `Admin` component to use the `authProvider` and `dataProvider` we created earlier.

```tsx
// src/Admin.tsx

import { OpenApiAdmin } from '@api-platform/admin';
import authProvider from "./authProvider";
import dataProvider from "./dataProvider";
import { API_DOCS_PATH, API_ENTRYPOINT_PATH } from "./config/api";

export default () => (
  <OpenApiAdmin
    entrypoint={API_ENTRYPOINT_PATH}
    docEntrypoint={API_DOCS_PATH}
    dataProvider={dataProvider}
    authProvider={authProvider}
  />
);
```
