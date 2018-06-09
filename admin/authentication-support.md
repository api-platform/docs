# Authentication Support

Authentication can easily be handled when using the API Platform's admin library.
In the following section, we will assume [the API is secured using JWT](https://api-platform.com/docs/core/jwt), but the
process is similar for other authentication mechanisms. The `login_uri` is the full URI to the route specified by the `firewalls.login.json_login.check_path` config in the [JWT documentation](https://api-platform.com/docs/core/jwt).

The first step is to create a client to handle the authentication process:

```javascript
// src/authClient.js
import { AUTH_LOGIN, AUTH_LOGOUT, AUTH_ERROR, AUTH_CHECK } from 'react-admin';

// Change this to be your own login check route.
const login_uri = 'https://demo.api-platform.com/login_check';

export default (type, params) => {
  switch (type) {
    case AUTH_LOGIN:
      const { username, password } = params;
      const request = new Request(`${login_uri}`, {
        method: 'POST',
        body: JSON.stringify({ email: username, password }),
        headers: new Headers({ 'Content-Type': 'application/json' }),
      });

      return fetch(request)
        .then(response => {
          if (response.status < 200 || response.status >= 300) throw new Error(response.statusText);

          return response.json();
        })
        .then(({ token }) => {
          localStorage.setItem('token', token); // The JWT token is stored in the browser's local storage
          window.location.replace('/');
        });

    case AUTH_LOGOUT:
      localStorage.removeItem('token');
      break;

    case AUTH_ERROR:
      if (401 === params.status || 403 === params.status) {
        localStorage.removeItem('token');

        return Promise.reject();
      }
      break;

    case AUTH_CHECK:
      return localStorage.getItem('token') ? Promise.resolve() : Promise.reject();

      default:
          return Promise.resolve();
  }
}
```

Then, configure the `Admin` component to use the authentication client we just created:

```javascript
import React from 'react';
import parseHydraDocumentation from '@api-platform/api-doc-parser/lib/hydra/parseHydraDocumentation';
import { HydraAdmin, hydraClient, fetchHydra as baseFetchHydra } from '@api-platform/admin';
import authClient from './authClient';
import { Redirect } from 'react-router-dom';

const entrypoint = 'https://demo.api-platform.com'; // Change this by your own entrypoint
const fetchHeaders = {'Authorization': `Bearer ${window.localStorage.getItem('token')}`};
const fetchHydra = (url, options = {}) => baseFetchHydra(url, {
    ...options,
    headers: new Headers(fetchHeaders),
});
const hydraClient = api => hydraClient(api, fetchHydra);
const apiDocumentationParser = entrypoint => parseHydraDocumentation(entrypoint, { headers: new Headers(fetchHeaders) })
    .then(
        ({ api }) => ({ api }),
        (result) => {
            switch (result.status) {
                case 401:
                    return Promise.resolve({
                        api: result.api,
                        customRoutes: [{
                            props: {
                                path: '/',
                                render: () => <Redirect to={`/login`}/>,
                            },
                        }],
                    });

                default:
                    return Promise.reject(result);
            }
        },
    );

export default props => (
    <HydraAdmin
        apiDocumentationParser={apiDocumentationParser}
        authClient={authClient}
        entrypoint={entrypoint}
        dataProvider={hydraClient}
    />
);
```

Refer to [the chapter dedicated to authentication in the React Admin documentation](https://marmelab.com/react-admin/Authentication.html)
for more information.
