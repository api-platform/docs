# Getting Started

## Installation

You'll need to install the skeleton and the library.

Start by installing [the Yarn package manager](https://yarnpkg.com/) ([NPM](https://www.npmjs.com/) is also supported) and
the [Create React App](https://facebook.github.io/create-react-app/) tool.

Then, create a new React application for your admin:

    $ create-react-app my-admin

Now, go to the newly created `my-admin` directory:

    $ cd my-admin

Finally, install the `@api-platform/admin` library:

    $ yarn add @api-platform/admin

## Creating the Admin

Edit the `src/App.js` file the following way:

```javascript
import React from 'react';
import { HydraAdmin } from '@api-platform/admin';

// Replace with your own API entrypoint: if https://example.com/api/books is the path
// to the collection of book resources, then the entrypoint is https://example.com/api
export default () => <HydraAdmin entrypoint="https://demo.api-platform.com"/>;
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

Note: if you don't want to hardcode the API URL, you can [use an environment variable](https://facebook.github.io/create-react-app/docs/adding-custom-environment-variables).

## Customizing the Admin

The API Platform's admin parses the Hydra documentation exposed by the API and transforms it to an object data structure. This data structure can be tailored to add, remove or customize resources and properties. To do so, we can leverage the `AdminBuilder` component provided by the library. It's a lower level component than the `HydraAdmin` one we used in the previous example. It allows to access the object storing the structure of admin's screens.

### Using Custom Components

In the following example, we change components used for the `description` property of the `books` resource to ones accepting HTML (respectively `RichTextField` that renders HTML markup and `RichTextInput`, a WYSIWYG editor).
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

    description.field = props => (
      <RichTextField {...props} source="description" />
    );

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
The `input` property allows to set the component used to render the input used in create and edit screens.

Any [field](https://marmelab.com/react-admin/Fields.html) or [input](https://marmelab.com/react-admin/Inputs.html) provided by the React Admin library can be used.

To go further, take a look to the "[Including react-admin on another React app](https://marmelab.com/react-admin/CustomApp.html)" documentation page of React Admin to learn how to use directly redux, react-router, and redux-saga along with components provided by this library.

### Managing Files and Images

In the following example, we will:
* find every [ImageObject](http://schema.org/ImageObject) resource. For each [contentUrl](http://schema.org/contentUrl) field, we will use [ImageField](https://marmelab.com/react-admin/Fields.html#imagefield) as `field` and [ImageInput](https://marmelab.com/react-admin/Inputs.html#imageinput) as `input`.
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
    const description = books.fields.find(({ name }) => 'description' === name);

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

            field.field = props => (
              <ImageField {...props} source={`${field.name}.src`} />
            );

            field.input = (
              <ImageInput accept="image/*" key={field.name} multiple={false} source={field.name}>
                <ImageField source="src"/>
              </ImageInput>
            );

            field.normalizeData = value => {
              if (value && value.rawFile instanceof File) {
                const body = new FormData();
                body.append('file', value.rawFile);

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

Example to add a minLength validator on the `description` field:

```javascript
import React, { Component } from 'react';
import { minLength } from 'react-admin';
import RichTextInput from 'ra-input-rich-text';
import { AdminBuilder, hydraClient } from '@api-platform-admin';
import parseHydraDocumentation from '@api-platform/api-doc-parser/lib/hydra/parseHydraDocumentation';

const entrypoint = 'https://demo.api-platform.com';

export default class extends Component {
  state = { api: null };

  componentDidMount() {
    parseHydraDocumentation(entrypoint).then(({ api }) => {
      const books = api.resources.find(({ name }) => 'books' === name);

      const description = books.fields.find(({ name }) => 'description' === name)
      description.input = props => (
        <RichTextInput source={description.name} label="Description" validate={minLength(30)} {...props} />
      )

      this.setState({ api });
    });
  }

  render() {
    if (null === this.state.api) return <div>Loading...</div>;

    return <AdminBuilder api={ this.state.api } dataProvider={ hydraClient(this.state.api) }/>
  }
}
```

### Show the Names of your Entities Instead of their IRIs

When you install API Platform Admin, you might see objects being referred as their IRIs instead of the name you would expect to see. This is because the component looks for this information in the Hydra data.

To configure which property should be shown to represent your entity, you have to include the following line in the docblock preceding your property:

```php
// api/src/Entity/Book.php

/**
 * @ApiProperty(iri="http://schema.org/name")
 */
private $name;
```

Besides, it is also possible to use the documentation to customize some fields automatically while configuring the semantics of your data.

You can use the `http://schema.org/email` and `http://schema.org/url` properties to create an `EmailField` and an `UrlField`, respectively.

### Using the Hydra Data Provider Directly with react-admin

By default, the `HydraAdmin` component shipped with API Platform Admin will generate a convenient admin interface for every resource and every property exposed by the API. But sometimes, you may prefer having full control over the generated admin.

To do so, an alternative approach is [to configure every react-admin component manually](https://marmelab.com/react-admin/Tutorial.html) instead of letting the library generate them, but to still leverage the built-in Hydra [data provider](https://marmelab.com/react-admin/DataProviders.html):

```javascript
// admin/src/App.js

import React, { Component } from 'react';
import { Admin, Resource } from 'react-admin';
import parseHydraDocumentation from '@api-platform/api-doc-parser/lib/hydra/parseHydraDocumentation';
import { hydraClient, fetchHydra as baseFetchHydra  } from '@api-platform/admin';
import authProvider from './authProvider';
import { Redirect } from 'react-router-dom';
import { createMuiTheme } from '@material-ui/core/styles';
import Layout from './Component/Layout';
import { UserShow } from './Components/User/Show';
import { UserEdit } from './Components/User/Edit';
import { UserCreate } from './Components/User/Create';
import { UserList } from './Components/User/List';

const theme = createMuiTheme({
    palette: {
        type: 'light'
    },
});

const entrypoint = process.env.REACT_APP_API_ENTRYPOINT;
const fetchHeaders = {'Authorization': `Bearer ${window.localStorage.getItem('token')}`};
const fetchHydra = (url, options = {}) => baseFetchHydra(url, {
    ...options,
    headers: new Headers(fetchHeaders),
});
const dataProvider = api => hydraClient(api, fetchHydra);
const apiDocumentationParser = entrypoint => parseHydraDocumentation(entrypoint, { headers: new Headers(fetchHeaders) })
    .then(
        ({ api }) => ({api}),
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

export default class extends Component {
    state = { api: null };

    componentDidMount() {
        apiDocumentationParser(entrypoint).then(({ api }) => {
            this.setState({ api });
        }).catch((e) => {
            console.log(e);
        });
    }

    render() {
        if (null === this.state.api) return <div>Loading...</div>;
        return (
            <Admin api={ this.state.api }
                   apiDocumentationParser={ apiDocumentationParser }
                   dataProvider= { dataProvider(this.state.api) }
                   theme={ theme }
                   appLayout={ Layout }
                   authProvider={ authProvider }          
            >                
                <Resource name="users" list={ UserList } create={ UserCreate } show={ UserShow } edit={ UserEdit } title="Users"/>
            </Admin>
        )
    }
}
```

And accordingly create files `Show.js`, `Create.js`, `List.js`, `Edit.js`
in the `admin/src/Component/User` directory:

```javascript
// admin/src/Component/User/Create.js

import React from 'react';
import { Create, SimpleForm, TextInput, email, required, ArrayInput, SimpleFormIterator } from 'react-admin';

export const UserCreate = (props) => (
    <Create { ...props }>
        <SimpleForm>
            <TextInput source="email" label="Email" validate={ email() } />
            <TextInput source="plainPassword" label="Password" validate={ required() } />
            <TextInput source="name" label="Name"/>
            <TextInput source="phone" label="Phone"/>
            <ArrayInput source="roles" label="Roles">
                <SimpleFormIterator>
                    <TextInput />
                </SimpleFormIterator>
            </ArrayInput>
        </SimpleForm>
    </Create>
);

```

```javascript
// admin/src/Component/User/Edit.js

import React from 'react';
import { Edit, SimpleForm, DisabledInput, TextInput, email, ArrayInput, SimpleFormIterator, DateInput } from 'react-admin';

export const UserEdit = (props) => (
    <Edit {...props}>
        <SimpleForm>
            <DisabledInput source="originId" label="ID"/>
            <TextInput source="email" label="Email" validate={ email() } />
            <TextInput source="name" label="Name"/>
            <TextInput source="phone" label="Phone"/>
            <ArrayInput source="roles" label="Roles">
                <SimpleFormIterator>
                    <TextInput />
                </SimpleFormIterator>
            </ArrayInput>
            <DateInput disabled source="createdAt" label="Date"/>
        </SimpleForm>
    </Edit>
);
```

```javascript
// admin/src/Component/User/List.js

import React from 'react';
import { List, Datagrid, TextField, EmailField, DateField, ShowButton, EditButton } from 'react-admin';
import { CustomPagination } from '../Pagination/CustomPagination';

export const UserList = (props) => (
    <List {...props} title="Users" pagination={ <CustomPagination/> }  perPage={ 30 }>
        <Datagrid>
            <TextField source="originId" label="ID"/>
            <EmailField source="email" label="Email" />
            <TextField source="name" label="Name"/>
            <TextField source="phone" label="Phone"/>
            <DateField source="createdAt" label="Date"/>
            <ShowButton />
            <EditButton />
        </Datagrid>
    </List>
);
```

```javascript
// admin/src/Component/User/Show.js
import React from 'react';
import { Show, SimpleShowLayout, TextField, DateField, EmailField } from 'react-admin';

export const UserShow = (props) => (
    <Show { ...props }>
        <SimpleShowLayout>
            <TextField source="originId" label="ID"/>
            <EmailField source="email" label="Email" />
            <TextField source="name" label="Name"/>
            <TextField source="phone" label="Phone"/>
            <DateField source="createdAt" label="Date"/>
        </SimpleShowLayout>
    </Show>
);
```
