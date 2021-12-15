# Authentication Support

API Platform Admin delegates the authentication support to React Admin.
Refer to [the chapter dedicated to authentication in the React Admin documentation](https://marmelab.com/react-admin/Authentication.html)
for more information.

In short, you have to tweak the data provider and the api documentation parser like this:

```typescript
// pwa/pages/admin/index.tsx

import Head from "next/head";
import { Redirect, Route } from "react-router-dom";
import { hydraDataProvider as baseHydraDataProvider, fetchHydra as baseFetchHydra, useIntrospection } from "@api-platform/admin";
import parseHydraDocumentation from "@api-platform/api-doc-parser/lib/hydra/parseHydraDocumentation";
import authProvider from "utils/authProvider";
import { ENTRYPOINT } from "config/entrypoint";

const getHeaders = () => localStorage.getItem("token") ? {
  Authorization: `Bearer ${localStorage.getItem("token")}`,
} : {};
const fetchHydra = (url, options = {}) =>
  baseFetchHydra(url, {
    ...options,
    headers: getHeaders,
  });
const RedirectToLogin = () => {
  const introspect = useIntrospection();

  if (localStorage.getItem("token")) {
    introspect();
    return <></>;
  }
  return <Redirect to="/login" />;
};
const apiDocumentationParser = async () => {
  try {
    const { api } = await parseHydraDocumentation(ENTRYPOINT, { headers: getHeaders });
    return { api };
  } catch (result) {
    if (result.status !== 401) {
      throw result;
    }

    // Prevent infinite loop if the token is expired
    localStorage.removeItem("token");

    return {
      api: result.api,
      customRoutes: [
        <Route key="/" path="/" component={RedirectToLogin} />
      ],
    };
  }
};
const dataProvider = baseHydraDataProvider({
  entrypoint: ENTRYPOINT,
  httpClient: fetchHydra,
  apiDocumentationParser,
});

const AdminLoader = () => {
  if (typeof window !== "undefined") {
    const { HydraAdmin } = require("@api-platform/admin");
    return <HydraAdmin dataProvider={dataProvider} authProvider={authProvider} entrypoint={window.origin} />;
  }

  return <></>;
};

const Admin = () => (
  <>
    <Head>
      <title>API Platform Admin</title>
    </Head>

    <AdminLoader />
  </>
);
export default Admin;
```

For the implementation of the auth provider, you can find a working example in the [API Platform's demo application](https://github.com/api-platform/demo/blob/main/pwa/utils/authProvider.tsx).
