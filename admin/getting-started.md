# Getting Started

## Installation

Install the skeleton and the library:

Start by installing [the Yarn package manager](https://yarnpkg.com/) ([NPM](https://www.npmjs.com/) is also supported) and
the [Create React App](https://github.com/facebookincubator/create-react-app) tool.

Then, create a new React application for your admin:

    $ create-react-app my-admin

Then, every files to edit and commands to launch will be in the new created my-admin directory.

    $ cd my-admin

React and React DOM will be directly provided as dependencies of Admin On REST. As having different versions of React
causes issues, remove `react` and `react-dom` from the `dependencies` section of the generated `package.json` file:

```patch
-    "react": "^15.6.1",
-    "react-dom": "^15.6.1"
```

Finally, install the `@api-platform/admin` library:

    $ yarn add @api-platform/admin

## Creating the Admin

Edit the `src/App.js` file like the following:

```javascript
import React from 'react';
import { HydraAdmin } from '@api-platform/admin';

export default () => <HydraAdmin entrypoint="https://demo.api-platform.com"/>; // Replace with your own API entrypoint
```

Replace the domain by your own domain in the `cors_allow_origin` param of `app/config/parameters.yml` symfony file. Ex: `http://localhost:3000` by default.

Clear cache of symfony for the production environment to take care of this changes:

    $ docker-compose exec app bin/console cache:clear --env=prod

Your new administration interface is ready! Type `yarn start` to try it!

Note: if you don't want to hardcode the API URL, you can [use an environment variable](https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md#adding-custom-environment-variables).

## Customizing the Admin

The API Platform's admin parses the Hydra documentation exposed by the API and transforms it to an object data structure. This data structure can be customized to add, remove or customize resources and properties. To do so, we can leverage the `AdminBuilder` component provided by the library. It's a lower level component than the `HydraAdmin` one we used in the previous example. It allows to access to the object storing the structure of admin's screens.

### Using Custom Components

In the following example, we change components used for the `description` property of the `books` resource to ones accepting HTML (respectively `RichTextField` that renders HTML markup and `RichTextInput`, a WYSWYG editor).
(To use the `RichTextInput`, the `aor-rich-text-input` package is must be installed: `yarn add aor-rich-text-input`).

```javascript
import React from 'react';
import { RichTextField } from 'admin-on-rest';
import RichTextInput from 'aor-rich-text-input';
import { HydraAdmin } from '@api-platform/admin';
import parseHydraDocumentation from '@api-platform/api-doc-parser/lib/hydra/parseHydraDocumentation';

const entrypoint = 'https://demo.api-platform.sh';

const apiDocumentationParser = entrypoint => parseHydraDocumentation(entrypoint)
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

export default (props) => (
  <HydraAdmin apiDocumentationParser={apiDocumentationParser} entrypoint={entrypoint}/>
);
```

The `fieldComponent` property of the `Field` class allows to set the component used to render a property in list and show screens.
The `inputComponent` property allows to set the component to use to render the input used in create and edit screens.

Any [field](https://marmelab.com/admin-on-rest/Fields.html) or [input](https://marmelab.com/admin-on-rest/Inputs.html) provided by the Admin On Rest library can be used.

To go further, take a look to the "[Including admin-on-rest on another React app](https://marmelab.com/admin-on-rest/CustomApp.html)" documentation page of Admin On Rest to learn how to use directly redux, react-router, and redux-saga along with components provided by this library.

### Managing Files and Images

In the following example, we will:
* find every [ImageObject](http://schema.org/ImageObject) resources. For each [contentUrl](http://schema.org/contentUrl) fields, we will use [ImageField](https://marmelab.com/admin-on-rest/Fields.html#imagefield) as `field` and [ImageInput](https://marmelab.com/admin-on-rest/Inputs.html#imageinput) as `input`.
* [ImageInput](https://marmelab.com/admin-on-rest/Inputs.html#imageinput) will return a [File](https://developer.mozilla.org/en/docs/Web/API/File) instance. In this example, we will send a multi-part form data to a special action (`https://demo.api-platform.com/images/upload`). The action will return the ID of the uploaded image. We will "replace" the [File](https://developer.mozilla.org/en/docs/Web/API/File) instance by the ID in `normalizeData`.
* As `contentUrl` fields will return a string, we have to convert Hydra data to AOR data. This action will be done by `denormalizeData`.

```javascript
import { FunctionField, ImageField, ImageInput } from 'admin-on-rest/lib/mui';
import React from 'react';
import { RichTextField } from 'admin-on-rest';
import RichTextInput from 'aor-rich-text-input';
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

export default (props) => (
  <HydraAdmin apiDocumentationParser={myApiDocumentationParser} entrypoint={entrypoint} />
);
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

    return <AdminBuilder api={this.state.api} restClient={hydraClient(entrypoint)}/>
  }
}
```
