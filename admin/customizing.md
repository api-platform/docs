# Customizing the Admin

## Preparing your App

```javascript
import React from 'react';
import { HydraAdmin, replaceResources } from '@api-platform/admin';
import parseHydraDocumentation from '@api-platform/api-doc-parser/lib/hydra/parseHydraDocumentation';
import { greetingsNameInput } from './components/greetings/inputs.js';
import { greetingsNameField } from './components/greetings/fields.js';
import BookList from './components/books/list.js';
import BookCreate from './components/books/create.js';
import BookEdit from './components/books/edit.js';
import { booksDescriptionInput } from './components/books/inputs.js';

const entrypoint = process.env.REACT_APP_API_ENTRYPOINT;

const greetings = {
  name: 'greetings',
  fields: [
    {
      name: 'name',
      input: greetingsNameInput,
      field: greetingsNameField,
    }
  ],
  listFields: [],
};

const books = {
  name: 'books',
  list: BookList,
  create: BookCreate,
  edit: BookEdit,
  fields: [
    {
      name: 'description',
      input: booksDescriptionInput,
    }
  ]
};

const newResources = [
  greetings,
  books,
];

const myApiDocumentationParser = entrypoint => parseHydraDocumentation(entrypoint)
  .then(({ api }) => {
    replaceResources(api.resources, newResources);
    return { api };
  })
;

export default () => <HydraAdmin apiDocumentationParser={myApiDocumentationParser} entrypoint={entrypoint} />;
```

All you have to do is to provide a collection of objects (the `newResources` variable).
The value of the `name` property must match the resource name you want to customize, or it will be ignored.
Other available properties will be explained further.

## Customizing Inputs

```javascript
import React from 'react';
import { TextInput } from 'react-admin';

const myInput = props => (
  <TextInput label="My new label" {...props} />
);

const greetings = {
  name: 'greetings',
  fields: [
    {
      name: 'name',
      input: myInput,
    },
  ],
};

export default [
  greetings,
];
```

That's it! Our custom `TextInput` component will now be used in all forms to edit the `name` property of the `greeting` resource.
In this example, we are reusing an `Input` component provided by `react-admin`, but you can use any component you want as long as you respect [the signature expected by react-admin](https://marmelab.com/react-admin/Inputs.html).

## Customizing Fields

```javascript
import React from 'react';
import { UrlField } from 'react-admin';

const myField = props => (
  <UrlField {...props} />
);

const greetings = {
  name: 'greetings',
  fields: [
    {
      name: 'url',
      field: myField,
    },
  ],
};

export default [
  greetings,
];
```

That's it! Our custom `myField` component will now be used to display the resource.
In this example, we are reusing a `Field` component provided by `react-admin`, but you can use any component you want as long as you respect [the signature expected by react-admin](https://marmelab.com/react-admin/Fields.html).

## "Free" Mode

If you want to fully customize the admin, here is how you can do it:

```javascript
import React from 'react';

const GreetingList = props => <p>Yay! I can do what I want!</p>;
const GreetingCreate = props => <p>Yay! I can do what I want!</p>;
const GreetingEdit = props => <p>Yay! I can do what I want!</p>;

const greetings = {
  name: 'greetings',
  list: GreetingList,
  create: GreetingCreate,
  edit: GreetingEdit,
};

export default [
  greetings,
];
```

## Reusing the Default Layout

Most of the time you want to keep the default layout and just customize what is inside, here is how to do it:

### List

```javascript
import React from 'react';
import { List, Datagrid } from 'react-admin';

const GreetingList = props => {
  const getField = fieldName => {
    const {options: {resource: {fields}}} = props;

    return fields.find(resourceField => resourceField.name === fieldName) ||
      null;
  };

  const displayField = fieldName => {
    const {options: {api, fieldFactory, resource}} = props;

    const field = getField(fieldName);

    if (field === null) {
      return;
    }

    return fieldFactory(field, {api, resource});
  };

  return (
    <List {...props}>
      <Datagrid>
        {displayField('name')}
      </Datagrid>
    </List>
  );
};

const greetings = {
  name: 'greetings',
  list: GreetingList,
};

export default [
  greetings,
];
```

### Create

#### Customizing the Form Layout

```javascript
import React from 'react';
import { Create, SimpleForm } from 'react-admin';
import { getResourceField } from '@api-platform/admin/lib/docsUtils';

const GreetingCreate = props => {
  const {options: {inputFactory, resource}} = props;

  return (
    <Create {...props}>
      <SimpleForm>
        <div className="custom-grid">
          <div className="column">
            {inputFactory(getResourceField(resource, 'name'))}
          </div>
          <div className="column">
            {inputFactory(getResourceField(resource, 'description'))}
          </div>
        </div>
      </SimpleForm>
    </Create>
  );
};

export default [
  greetings,
];
```

This way, we have been reusing most of the default behavior, but we managed to had a custom grid. This could also be a way to customize the fields order, and many more things you could think of.

#### Dynamic Display

If you want to have a form with dynamic display, just use a connected component like this one:

```javascript
import React from 'react';
import { connect } from 'react-redux';
import { formValueSelector } from 'redux-form';
import { getResourceField } from '@api-platform/admin/lib/docsUtils';
import { Create, SimpleForm } from 'react-admin';

const GreetingCreateView = props => {
  const {options: {inputFactory, resource}, formValueName} = props;

  return (
    <Create {...props}>
      <SimpleForm>
        {inputFactory(getResourceField(resource, 'name'))}
        {formValueName && (
          inputFactory(getResourceField(resource, 'description'))
        )}
      </SimpleForm>
    </Create>
  );
};

const mapStateToProps = state => ({
  formValueName: formValueSelector('record-form')(state, 'name'),
});

const GreetingCreate = connect(mapStateToProps)(GreetingCreateView);

const greetings = {
  name: 'greetings',
  create: GreetingCreate,
};

export default [
  greetings,
];
```

### Edit

```javascript
import React from 'react';
import { Edit, SimpleForm } from 'react-admin';
import { getResourceField } from '@api-platform/admin/lib/docsUtils';

const GreetingEdit = props => {
  const {options: {inputFactory, resource}} = props;

  return (
    <Edit {...props}>
      <SimpleForm>
        <div className="custom-grid">
          <div className="column">
            {inputFactory(getResourceField(resource, 'name'))}
          </div>
          <div className="column">
            {inputFactory(getResourceField(resource, 'description'))}
          </div>
        </div>
      </SimpleForm>
    </Edit>
  );
};

export default [
  greetings,
];
```

In this example, we have been able to customize the template to add a custom grid, but you could do more, have a look at the `Create`part above to see more examples.
