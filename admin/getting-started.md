# Getting Started

## Installation

If you use the [API Platform Distribution](../distribution/), API Platform Admin is already installed, you can skip this installation guide.

Otherwise, all you need to install API Platform Admin is a JavaScript package manager. We recommend [Yarn](https://yarnpkg.com/) ([npm](https://www.npmjs.com/) is also supported).

If you don't have an existing React Application, create one using [Create React App](https://create-react-app.dev/):

```console
yarn create react-app my-admin
```

Go to the directory of your project:

```console
cd my-admin
```

Finally, install the `@api-platform/admin` library:

```console
yarn add @api-platform/admin
```

## Creating the Admin

To initialize API Platform Admin, register it in your application.
For instance, if you used Create React App, replace the content of `src/App.js` by:

```javascript
import { HydraAdmin } from "@api-platform/admin";

// Replace with your own API entrypoint
// For instance if https://example.com/api/books is the path to the collection of book resources, then the entrypoint is https://example.com/api
export default () => (
  <HydraAdmin entrypoint="https://demo.api-platform.com" />
);
```

Be sure to make your API send proper [CORS HTTP headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) to allow
the admin's domain to access it.

To do so, if you use the API Platform Distribution, update the value of the `CORS_ALLOW_ORIGIN` parameter in `.env` (it will be set to `^https?://localhost:?[0-9]*$`
by default).

If you use a custom installation of Symfony and [API Platform Core](../core/), you will need to adjust the [NelmioCorsBundle configuration](https://github.com/nelmio/NelmioCorsBundle#configuration) to expose the `Link` HTTP header and to send proper CORS headers on the route under which the API will be served (`/api` by default).
Here is a sample configuration:

```yaml
# config/packages/nelmio_cors.yaml

nelmio_cors:
    paths:
        '^/api/':
            origin_regex: true
            allow_origin: ['^http://localhost:[0-9]+'] # You probably want to change this regex to match your real domain
            allow_methods: ['GET', 'OPTIONS', 'POST', 'PUT', 'PATCH', 'DELETE']
            allow_headers: ['Content-Type', 'Authorization']
            expose_headers: ['Link']
            max_age: 3600
```

Clear the cache to apply this change:

```console
docker-compose exec php \
    bin/console cache:clear --env=prod
```

Your new administration interface is ready! Type `yarn start` to try it!

Note: if you don't want to hardcode the API URL, you can [use an environment variable](https://create-react-app.dev/docs/adding-custom-environment-variables).
