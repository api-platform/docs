# Getting Started

## Installation

Install the skeleton and the library:

Start by installing [the Yarn package manager](https://yarnpkg.com/) ([NPM](https://www.npmjs.com/) is also supported) and
the [Create React App](https://github.com/facebookincubator/create-react-app) tool.

Then, create a new React application for your admin:

    $ create-react-app my-admin

Now, go to the newly created `my-admin` directory:

    $ cd my-admin

Finally, install the `@api-platform/admin` library:

    $ yarn add @api-platform/admin

## Creating the Admin

Edit the `src/App.js` file like the following:

```javascript
import React from 'react';
import { HydraAdmin } from '@api-platform/admin';

export default () => <HydraAdmin entrypoint="https://demo.api-platform.com"/>; // Replace with your own API entrypoint
```

Be sure to make your API send proper [CORS HTTP headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) to allow
the admin's domain to access it.
To do so, update the value of the `CORS_ALLOW_ORIGIN` parameter in `api/.env` (it will be set to `^https?://localhost:?[0-9]*$`
by default).

If you're not using the API Platform distribution, you will need to adjust the NelmioCorsBundle configuration to expose the `Link` HTTP header and to send proper CORS headers on the route under which the API will be served (`/api` by default).
Here is a sample configuration (if you use the API Platform distribution, you can skip this step):

```yaml
# config/packages/nelmio-cors.yaml

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

Note: if you don't want to hardcode the API URL, you can [use an environment variable](https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md#adding-custom-environment-variables).

## Customizing the Admin

The API Platform's admin parses the Hydra documentation exposed by the API and transforms it to an object data structure. This data structure can be customized to add, remove or customize resources and properties. To do so, we can leverage the `AdminBuilder` component provided by the library. It's a lower level component than the `HydraAdmin` one we used in the previous example. It allows to access to the object storing the structure of admin's screens.

### Using Custom Components

In the following example, we change components used for the `description` property of the `books` resource to ones accepting HTML (respectively `RichTextField` that renders HTML markup and `RichTextInput`, a WYSWYG editor).
(To use the `RichTextInput`, the `ra-input-rich-text` package is must be installed: `yarn add ra-input-rich-text`).

```javascript
import React from 'react';
import { RichTextField } from 'react-admin';
import RichTextInput from 'ra-input-rich-text';
import { HydraAdmin } from '@api-platform/admin';
import parseHydraDocumentation from '@api-platform/api-doc-parser/lib/hydra/parseHydraDocumentation';

const entrypoint = 'https://demo.api-platform.com';

const myApiDocumentationParser = entrypoint => parseHydraDocumentation(entrypoint)
  .then( ({ api }) => {
    const books = api.resources.find(({ name }) => 'books' === name);
    const description = books.fields.find(f => 'description' === f.name);

    description.input = props => (
      <RichTextInput {...props} source="description" />
    );

    description.input.defaultProps = {
      addField: true,
      addLabel: true
    };
    
    return { api };
  })
;

export default (props) => <HydraAdmin apiDocumentationParser={myApiDocumentationParser} entrypoint={entrypoint}/>;
```

The `field` property of the `Field` class allows to set the component used to render a property in list and show screens.
The `input` property allows to set the component to use to render the input used in create and edit screens.

Any [field](https://marmelab.com/react-admin/Fields.html) or [input](https://marmelab.com/react-admin/Inputs.html) provided by the React Admin library can be used.

To go further, take a look to the "[Including react-admin on another React app](https://marmelab.com/react-admin/CustomApp.html)" documentation page of React Admin to learn how to use directly redux, react-router, and redux-saga along with components provided by this library.

### Managing Files and Images

In the following example, we will:
* find every [ImageObject](http://schema.org/ImageObject) resources. For each [contentUrl](http://schema.org/contentUrl) fields, we will use [ImageField](https://marmelab.com/react-admin/Fields.html#imagefield) as `field` and [ImageInput](https://marmelab.com/react-admin/Inputs.html#imageinput) as `input`.
* [ImageInput](https://marmelab.com/react-admin/Inputs.html#imageinput) will return a [File](https://developer.mozilla.org/en/docs/Web/API/File) instance. In this example, we will send a multi-part form data to a special action (`https://demo.api-platform.com/images/upload`). The action will return the ID of the uploaded image. We will "replace" the [File](https://developer.mozilla.org/en/docs/Web/API/File) instance by the ID in `normalizeData`.
* As `contentUrl` fields will return a string, we have to convert Hydra data to React Admin data. This action will be done by `denormalizeData`.

```javascript
import React from 'react';
import { FunctionField, ImageField, ImageInput, RichTextField } from 'react-admin';
import RichTextInput from 'ra-input-rich-text';
import { HydraAdmin } from '@api-platform/admin';
import parseHydraDocumentation from '@api-platform/api-doc-parser/lib/hydra/parseHydraDocumentation';

const entrypoint = 'https://demo.api-platform.com';

const myApiDocumentationParser = entrypoint => parseHydraDocumentation(entrypoint)
  .then( ({ api }) => {

    const books = api.resources.find(({ name }) => 'books' === name);
    const description = books.fields.find(f => 'description' === f.name);

    description.input = props => (
      <RichTextInput {...props} source="description" />
    );

    description.input.defaultProps = {
      addField: true,
      addLabel: true,
    };

    api.resources.map(resource => {
      if ('http://schema.org/ImageObject' === resource.id) {
        resource.fields.map(field => {
          if ('http://schema.org/contentUrl' === field.id) {
            field.denormalizeData = value => ({
              src: value
            });

            field.fieldComponent = (
              <FunctionField
                key={field.name}
                render={
                  record => (
                    <ImageField key={field.name} record={record} source={`${field.name}.src`}/>
                  )
                }
                source={field.name}
              />
            );

            field.inputComponent = (
              <ImageInput accept="image/*" key={field.name} multiple={false} source={field.name}>
                <ImageField source="src"/>
              </ImageInput>
            );

            field.normalizeData = value => {
              if (value[0] && value[0].rawFile instanceof File) {
                const body = new FormData();
                body.append('file', value[0].rawFile);

                return fetch(`${entrypoint}/images/upload`, { body, method: 'POST' })
                  .then(response => response.json());
              }

              return value.src;
            };
          }

          return field;
        });
      }

      return resource;
    });

    return { api };
  })
;

export default (props) => <HydraAdmin apiDocumentationParser={myApiDocumentationParser} entrypoint={entrypoint}/>;
```

__Note__: In this example, we choose to send the file via a multi-part form data, but you are totally free to use another solution (like `base64`). But keep in mind that multi-part form data is the most efficient solution.

### Using a Custom Validation Function or Inject Custom Props

You can use `fieldProps` and `inputProps` to respectively inject custom properties to fields and inputs generated by API
Platform Admin. This is particularly useful to add custom validation rules:

```javascript
import React, { Component } from 'react';
import { AdminBuilder, hydraClient } from 'api-platform-admin';
import parseHydraDocumentation from 'api-doc-parser/lib/hydra/parseHydraDocumentation';

const entrypoint = 'https://demo.api-platform.com';

export default class extends Component {
  state = {api: null};

  componentDidMount() {
    parseHydraDocumentation(entrypoint).then( ({ api }) =>  => {
      const books = api.resources.find(r => 'books' === r.name);

      books.writableFields.find(f => 'description' === f.name).inputProps = {
        validate: value => value.length >= 30 ? undefined : 'Minimum length: 30';
      };

      this.setState({api: api});
      
      return { api };
    });
  }

  render() {
    if (null === this.state.api) return <div>Loading...</div>;

    return <AdminBuilder api={this.state.api} dataProvider={hydraClient(entrypoint)}/>
  }
}
```
