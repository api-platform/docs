# Getting Started

## Installation

If you use the [API Platform Distribution](../distribution/), API Platform Admin is already installed, you can skip this installation guide.

Otherwise, all you need to install API Platform Admin is a JavaScript package manager. We recommend [Yarn](https://yarnpkg.com/) ([npm](https://www.npmjs.com/) is also supported).

If you don't have an existing React Application, create one using [Create React App](https://create-react-app.dev/):

    $ yarn create react-app my-admin

Go to the directory of your project:

    $ cd my-admin

Finally, install the `@api-platform/admin` library:

    $ yarn add @api-platform/admin

## Creating the Admin (React)

To initialize API Platform Admin, register it in your application.
For instance, if you used Create React App, replace the content of `src/App.js` by:

```javascript
import React from "react";
import { HydraAdmin } from "@api-platform/admin";

// Replace with your own API entrypoint
// For instance if https://example.com/api/books is the path to the collection of book resources, then the entrypoint is https://example.com/api
export default () => (
  <HydraAdmin entrypoint="https://demo.api-platform.com" />
);
```

Be sure to make your API send proper [CORS HTTP headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) to allow
the admin's domain to access it.

To do so, if you use the API Platform Distribution, update the value of the `CORS_ALLOW_ORIGIN` parameter in `api/.env` (it will be set to `^https?://localhost:?[0-9]*$`
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

    $ docker-compose exec php bin/console cache:clear --env=prod

Your new administration interface is ready! Type `yarn start` to try it!

Note: if you don't want to hardcode the API URL, you can [use an environment variable](https://create-react-app.dev/docs/adding-custom-environment-variables).

## Creating the Admin (React) and customize it with Twig

You can install the admin component with Composer by using:

    $ composer require api-admin
    
You also need to install the JavaScript libraries: 

    $ yarn add react react-dom @babel/preset-react
    
Don't forget to uncomment the following lines in your `webpack.config.js` file:

```javascript
.enableReactPreset()
.addEntry('admin', './assets/js/admin.js')
```

If you try to access to your `/admin` route, the new admin will show.
You can customize the render in `templates/admin.html.twig`. 
The file contains actually this : 
```twig
{% extends 'base.html.twig' %}

{% block title %}API Platform Admin{% endblock %}

{% block body %}<div id="api-platform-admin"></div>{% endblock %}

{% block javascripts %}
    <script type="text/plain" id="api-entrypoint">{{ path('api_entrypoint') }}</script>
    {{ encore_entry_script_tags('admin') }}
{% endblock %}
```
