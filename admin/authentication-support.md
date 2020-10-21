# Authentication Support

API Platform Admin delegates the authentication support to React Admin.
Refer to [the chapter dedicated to authentication in the React Admin documentation](https://marmelab.com/react-admin/Authentication.html)
for more information.

In short, you have to tweak data provider and api documentation parser, like this:

```javascript
// admin/src/App.js

import React from "react";
import { Redirect, Route } from "react-router-dom";
import { HydraAdmin, hydraDataProvider as baseHydraDataProvider, fetchHydra as baseFetchHydra } from "@api-platform/admin";
import parseHydraDocumentation from "@api-platform/api-doc-parser/lib/hydra/parseHydraDocumentation";
import authProvider from "./authProvider";

const entrypoint = process.env.REACT_APP_API_ENTRYPOINT;
const fetchHeaders = () => ({
  Authorization: `Bearer ${window.localStorage.getItem("token")}`,
});
const fetchHydra = (url, options = {}) =>
    localStorage.getItem("token")
        ? baseFetchHydra(url, {
              ...options,
              headers: new Headers(fetchHeaders()),
          })
        : baseFetchHydra(url, options);
const apiDocumentationParser = (entrypoint) =>
    parseHydraDocumentation(
        entrypoint,
        localStorage.getItem("token")
            ? { headers: new Headers(fetchHeaders()) }
            : {}
    ).then(
        ({ api }) => ({ api }),
        (result) => {
            if (result.status === 401) {
                // Prevent infinite loop if the token is expired
                localStorage.removeItem("token");

                return Promise.resolve({
                    api: result.api,
                    customRoutes: [
                        <Route path="/" render={() => {
                            return localStorage.getItem("token") ? window.location.reload() : <Redirect to="/login" />
                        }} />
                    ],
                });
            }

            return Promise.reject(result);
        },
    );
const dataProvider = baseHydraDataProvider(entrypoint, fetchHydra, apiDocumentationParser);

export default () => (
    <HydraAdmin
        dataProvider={ dataProvider }
        authProvider={ authProvider }
        entrypoint={ entrypoint }
    />
);
```
