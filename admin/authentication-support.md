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

### Api Documentation Parsing

Handle API documentation parsing and clear an expired token.

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

Initialize the data provider with custom headers and the documentation parser.

```typescript
const dataProvider = (setRedirectToLogin) =>
  baseHydraDataProvider({
    entrypoint: ENTRYPOINT,
    httpClient: fetchHydra,
    apiDocumentationParser: apiDocumentationParser(setRedirectToLogin),
  });
```

### Export Admin Component

Export the admin component, and track the users' authentication status.

```typescript
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

// *****
// Here put the code parts shown above
// *****

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

For the implementation of the admin conponent, you can find a working example in the [API Platform's demo application](https://github.com/api-platform/demo/blob/4.0/pwa/components/admin/Admin.tsx).
