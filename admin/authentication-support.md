# Authentication Support

API Platform Admin delegates the authentication support to React Admin.
Refer to [the chapter dedicated to authentication in the React Admin documentation](https://marmelab.com/react-admin/Authentication.html)
for more information.

## HydraAdmin

The authentication layer for [HydraAdmin component](https://api-platform.com/docs/admin/components/#hydra)
consists of a few parts, which need to be integrated together.

### Authentication

Add the Bearer token from `localStorage` to request headers.

```typescript
const getHeaders = () =>
  localStorage.getItem("token")
    ? { Authorization: `Bearer ${localStorage.getItem("token")}` }
    : {};
```

Extend the Hydra fetch function with custom headers for authentication.

```typescript
const fetchHydra = (url, options = {}) =>
  baseFetchHydra(url, {
    ...options,
    headers: getHeaders,
  });

```

### Login Redirection

Redirect users to a `/login` path, if no token is available in the `localStorage`.

```typescript
const RedirectToLogin = () => {
  const introspect = useIntrospection();

  if (localStorage.getItem("token")) {
    introspect();
    return <></>;
  }
  return <Navigate to="/login" />;
};
```

### API Documentation Parsing

Extend the `parseHydraDocumentaion` function from the [API Doc Parser library](https://github.com/api-platform/api-doc-parser)
to handle the documentation parsing. Customize it to clear
expired tokens when encountering unauthorized `401` response.

```typescript
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

### Data Provider

Initialize the hydra data provider with custom headers and the documentation parser.

```typescript
const dataProvider = (setRedirectToLogin) =>
  baseHydraDataProvider({
    entrypoint: ENTRYPOINT,
    httpClient: fetchHydra,
    apiDocumentationParser: apiDocumentationParser(setRedirectToLogin),
  });
```

### Export Admin Component

Export the Hydra admin component, and track the users' authentication status.

```typescript
// components/admin/Admin.tsx

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

// Auth, Parser, Provider calls
const getHeaders = () => {...};
const fetchHydra = (url, options = {}) => {...};
const RedirectToLogin = () => {...};
const apiDocumentationParser = (setRedirectToLogin) => async () => {...};
const dataProvider = (setRedirectToLogin) => {...};

const Admin = () => {
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
export default Admin;
```

### Additional Notes

For the implementation of the admin component, you can find a working example in the [API Platform's demo application](https://github.com/api-platform/demo/blob/4.0/pwa/components/admin/Admin.tsx).

## OpenApiAdmin

This section explains how to set up and customize the [OpenApiAdmin component](https://api-platform.com/docs/admin/components/#openapi) authentication layer.
It covers:
* Creating a custom HTTP Client
* Data and rest data provider configuration
* Implementation of an auth provider

### Data Provider & HTTP Client

Create a custom HTTP client to add authentication tokens to request headers.
Configure the data `ApiPlatformAdminDataProvider` data provider, and
inject the custom HTTP client into the [Simple REST Data Provider for React-Admin](https://github.com/Serind/ra-data-simple-rest).

**File:** `src/components/jsonDataProvider.tsx`
```typescript
const httpClient = async (url: string, options: fetchUtils.Options = {}) => {
    options.headers = new Headers({
        ...options.headers,
        Accept: 'application/json',
    }) as Headers;

    const token = getAccessToken();
    options.user = { token: `Bearer ${token}`, authenticated: !!token };

    return await fetchUtils.fetchJson(url, options);
};

const jsonDataProvider = openApiDataProvider({
  dataProvider: simpleRestProvider(API_ENTRYPOINT_PATH, httpClient),
  entrypoint: API_ENTRYPOINT_PATH,
  docEntrypoint: API_DOCS_PATH,
});
```

> [!NOTE]
> The `simpleRestProvider` provider expect the API to include a `Content-Range` header in the response.
> You can find more about the header syntax in the [Mozilla’s MDN documentation: Content-Range](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Range).
> 
> The `getAccessToken` function retrieves the JWT token stored in the browser.

### Authentication and Authorization

Create and export an `authProvider` object that handles authentication and authorization logic.

**File:** `src/components/authProvider.tsx`
```typescript
interface JwtPayload {
    exp?: number;
    iat?: number;
    roles: string[];
    username: string;
}

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
            id: "",
            fullName: decoded.username,
            avatar: "",
        });
    },
    getPermissions: () => Promise.resolve(""),
};

export default authProvider;
```

### Export OpenApiAdmin Component

**File:** `src/App.tsx`
```typescript
import {OpenApiAdmin} from '@api-platform/admin';
import authProvider from "./components/authProvider";
import jsonDataProvider from "./components/jsonDataProvider";
import {API_DOCS_PATH, API_ENTRYPOINT_PATH} from "./config/api";

export default () => (
  <OpenApiAdmin
    entrypoint={API_ENTRYPOINT_PATH}
    docEntrypoint={API_DOCS_PATH}
    dataProvider={jsonDataProvider}
    authProvider={authProvider}
  />
);
```
