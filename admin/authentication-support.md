# Authentication Support

Authentication can easily be handled when using the API Platform's admin library.
In the following section, we will assume [the API is secured using JWT](https://api-platform.com/docs/core/jwt), but the
process is similar for other authentication mechanisms.

The first step is to create a client to handle the authentication process:

```javascript
// src/authClient.js
import { AUTH_LOGIN, AUTH_LOGOUT, AUTH_ERROR } from 'admin-on-rest';

const entrypoint = 'https://demo.api-platform.com'; // Change this by your own entrypoint

export default (type, params) => {
  switch (type) {
    case AUTH_LOGIN:
      const { username, password } = params;
      const request = new Request(`${entrypoint}/login_check`, {
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
// src/Admin.js
import React, { Component } from 'react';
import { HydraAdmin, hydraClient, fetchHydra } from '@api-platform/admin';
import authClient from './authClient';

const entrypoint = 'https://demo.api-platform.com';

const fetchWithAuth = (url, options = {}) => {
  if (!options.headers) options.headers = new Headers({ Accept: 'application/ld+json' });

  options.headers.set('Authorization', `Bearer ${localStorage.getItem('token')}`);
  return fetchHydra(url, options);
};

class Admin extends Component {
  render() {
    return <HydraAdmin entrypoint={entrypoint} restClient={hydraClient(entrypoint, fetchWithAuth)} authClient={authClient}/>
  }
}

export default Admin;
```

Refer to [the chapter dedicated to authentication in the Admin On Rest documentation](https://marmelab.com/admin-on-rest/Authentication.html)
for more information.

Previous chapter: [Getting Started](getting-started.md)

Next chapter: [Handling Relations to Collections](handling-relations-to-collections.md)
