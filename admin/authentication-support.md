# Authentication Support

API Platform Admin delegates the authentication support to React Admin.
Refer to [the chapter dedicated to authentication in the React Admin documentation](https://marmelab.com/react-admin/Authentication.html)
for more information.

In short, you have to tweak data provider and api documentation parser, like this:

```javascript
// admin/src/App.js

import React from "react";
import { Redirect, Route } from "react-router-dom";
import { HydraAdmin, hydraDataProvider as baseHydraDataProvider, fetchHydra as baseFetchHydra, useIntrospection } from "@api-platform/admin";
import parseHydraDocumentation from "@api-platform/api-doc-parser/lib/hydra/parseHydraDocumentation";
import authProvider from "./authProvider";

const entrypoint = process.env.REACT_APP_API_ENTRYPOINT;
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
const apiDocumentationParser = async (entrypoint) => {
  try {
    const { api } = await parseHydraDocumentation(entrypoint, { headers: getHeaders });
    return { api };
  } catch (result) {
    if (result.status === 401) {
      // Prevent infinite loop if the token is expired
      localStorage.removeItem("token");

      return {
        api: result.api,
        customRoutes: [
          <Route path="/" component={RedirectToLogin} />
        ],
      };
    }

    throw result;
  }
};
const dataProvider = baseHydraDataProvider(entrypoint, fetchHydra, apiDocumentationParser);

export default () => (
  <HydraAdmin
    dataProvider={ dataProvider }
    authProvider={ authProvider }
    entrypoint={ entrypoint }
  />
);
```

For the implementation of the auth provider, you can find a working example in the [API Platform's demo application](https://github.com/api-platform/demo/blob/master/admin/src/authProvider.js).
