# Components Reference

## Resource Components

### AdminGuesser

`<AdminGuesser>` renders automatically an [`<Admin>` component](https://marmelab.com/react-admin/Admin.html) for resources exposed by a web API documented with any format supported by `@api-platform/api-doc-parser`.

**Tip:** For Hydra documented APIs, use the [`<HydraAdmin>` component](#hydraadmin) instead.

**Tip:** For OpenAPI documented APIs, use the [`<OpenApiAdmin>` component](#openapiadmin) instead.

It also creates a [schema analyzer](#hydra-schema-analyzer) context, where the `schemaAnalyzer` service (for getting information about the provided API documentation) is stored.

`<AdminGuesser>` renders all exposed resources by default, but you can choose what resource you want to render by passing [`<ResourceGuesser>` components](#resourceguesser) as children.

**Tip:** Deprecated resources are hidden by default, but you can add them back using an explicit `<ResourceGuesser>` component.

```tsx
// App.tsx
import { AdminGuesser, ResourceGuesser } from '@api-platform/admin';

const App = () => (
  <AdminGuesser dataProvider={dataProvider} authProvider={authProvider}>
    <ResourceGuesser
      name="books"
      list={BooksList}
      show={BooksShow}
      edit={BooksEdit}
      create={BooksCreate}
    />
    <ResourceGuesser name="authors" />
  </AdminGuesser>
);

export default App;
```

#### Props

| Name              | Type            | Value          | required | Description                                                                      |
| ----------------- | --------------- | -------------- | -------- | -------------------------------------------------------------------------------- |
| dataProvider      | object          | dataProvider   | yes      | the dataProvider to use to communicate with your API                             |
| schemaAnalyzer    | object          | schemaAnalyzer | yes      | retrieves resource type according to [Schema.org](https://schema.org) vocabulary |
| admin             | React component | -              | no       | React component to use to render the Admin                                       |
| includeDeprecated | boolean         | true or false  | no       | displays or not deprecated resources                                             |

`<AdminGuesser>` also accepts all props accepted by React Admin's [`<Admin>` component](https://marmelab.com/react-admin/Admin.html), such as [`theme`](https://marmelab.com/react-admin/Admin.html#theme), [`darkTheme`](https://marmelab.com/react-admin/Admin.html#darktheme), [`layout`](https://marmelab.com/react-admin/Admin.html#layout) and many others.

### ResourceGuesser

Based on React Admin [`<Resource>` component](https://marmelab.com/react-admin/Resource.html), `<ResourceGuesser>` provides the default component to render for each view: [`<CreateGuesser>`](#createguesser), [`<ListGuesser>`](#listguesser), [`<EditGuesser>`](#editguesser) and [`<ShowGuesser>`](#showguesser).

You can also pass your own component to use for any view, using the `create`, `list`, `edit` or `show` props.

```tsx
// App.tsx
import { AdminGuesser, ResourceGuesser } from '@api-platform/admin';

const App = () => (
  <AdminGuesser dataProvider={dataProvider} schemaAnalyzer={schemaAnalyzer}>
    {/* Uses the default guesser components for each CRUD view */}
    <ResourceGuesser name="books" />
    {/* Overrides only the list view */}
    <ResourceGuesser name="reviews" list={ReviewList} />
  </AdminGuesser>
);

export default App;
```

#### ResourceGuesser Props

| Name | Type   | Value | required | Description              |
| ---- | ------ | ----- | -------- | ------------------------ |
| name | string | -     | yes      | endpoint of the resource |

`<ResourceGuesser>` also accepts all props accepted by React Admin's [`<Resource>` component](https://marmelab.com/react-admin/Resource.html), such as [`recordRepresentation`](https://marmelab.com/react-admin/Resource.html#recordrepresentation), [`icon`](https://marmelab.com/react-admin/Resource.html#icon) or [`options`](https://marmelab.com/react-admin/Resource.html#options).

## Page Components

### ListGuesser

Based on React Admin [`<List>`](https://marmelab.com/react-admin/List.html), `<ListGuesser>` displays a list of records in a [`<Datagrid>`](https://marmelab.com/react-admin/Datagrid.html).

If no children are passed, it will display fields guessed from the schema.

```tsx
// BooksList.tsx
import { ListGuesser } from '@api-platform/admin';

export const BooksList = () => (
  /* Will display fields guessed from the schema */
  <ListGuesser />
);
```

It also accepts a list of fields as children. They can be either  [`<FieldGuesser>`](#fieldguesser) elements, or any [field component](https://marmelab.com/react-admin/Fields.html)
available in React Admin, such as [`<TextField>`](https://marmelab.com/react-admin/TextField.html), [`<DateField>`](https://marmelab.com/react-admin/DateField.html) or [`<ReferenceField>`](https://marmelab.com/react-admin/ReferenceField.html) for instance.

```tsx
// BooksList.tsx
import { FieldGuesser, ListGuesser } from '@api-platform/admin';
import { DateField, NumberField } from 'react-admin';

export const BooksList = () => (
  <ListGuesser>
      {/* FieldGuesser comes from API Platform Admin */}
      <FieldGuesser source="isbn" label="ISBN" />
      <FieldGuesser source="title" />
      <FieldGuesser source="author" />

      {/* DateField and NumberField come from React Admin */}
      <DateField source="publicationDate" />
      <NumberField source="reviews.length" label="Reviews" />
  </ListGuesser>
);
```

#### ListGuesser Props

`<ListGuesser>` accepts all props accepted by both React Admin [`<List>` component](https://marmelab.com/react-admin/List.html) and [`<Datagrid>` component](https://marmelab.com/react-admin/Datagrid.html).

For instance you can pass props such as [`filters`](https://marmelab.com/react-admin/List.html#filters-filter-inputs), [`sort`](https://marmelab.com/react-admin/List.html#sort) or [`pagination`](https://marmelab.com/react-admin/List.html#pagination).

### CreateGuesser

Displays a creation page for a single item. Uses React Admin [`<Create>`](https://marmelab.com/react-admin/Create.html) and [`<SimpleForm>`](https://marmelab.com/react-admin/SimpleForm.html) components.

If no children are passed, it will display inputs guessed from the schema.

```tsx
// BooksCreate.tsx
import { CreateGuesser } from '@api-platform/admin';

export const BooksCreate = () => (
  /* Will display inputs guessed from the schema */
  <CreateGuesser />
);
```

It also accepts a list of inputs as children, which can be either [`<InputGuesser>`](#inputguesser) elements, or any [input component](https://marmelab.com/react-admin/Inputs.html) available in React Admin, such as [`<TextInput>`](https://marmelab.com/react-admin/TextInput.html), [`<DateInput>`](https://marmelab.com/react-admin/DateInput.html) or [`<ReferenceInput>`](https://marmelab.com/react-admin/ReferenceInput.html) for instance.

```tsx
// BooksCreate.tsx
import { CreateGuesser, InputGuesser } from '@api-platform/admin';
import { DateInput, TextInput, required } from 'react-admin';

export const BooksCreate = () => (
  <CreateGuesser>
    {/* InputGuesser comes from API Platform Admin */}
    <InputGuesser source="isbn" label="ISBN" />
    <InputGuesser source="title" />
    <InputGuesser source="author" />

    {/* DateInput and TextInput come from React Admin */}
    <DateInput source="publicationDate" />
    <TextInput
      source="description"
      multiline
      validate={required()}
    />
  </CreateGuesser>
);
```

#### CreateGuesser Props

`<CreateGuesser>` accepts all props accepted by both React Admin [`<Create>` component](https://marmelab.com/react-admin/Create.html) and [`<SimpleForm>` component](https://marmelab.com/react-admin/SimpleForm.html).

For instance you can pass props such as [`redirect`](https://marmelab.com/react-admin/Create.html#redirect), [`defaultValues`](https://marmelab.com/react-admin/SimpleForm.html#defaultvalues) or [`warnWhenUnsavedChanges`](https://marmelab.com/react-admin/SimpleForm.html#warnwhenunsavedchanges).

### EditGuesser

Displays an edition page for a single item. Uses React Admin [`<Edit>`](https://marmelab.com/react-admin/Edit.html) and [`<SimpleForm>`](https://marmelab.com/react-admin/SimpleForm.html) components.

If no children are passed, it will display inputs guessed from the schema.

```tsx
// BooksEdit.tsx
import { EditGuesser } from '@api-platform/admin';

export const BooksEdit = () => (
  /* Will display inputs guessed from the schema */
  <EditGuesser />
);
```

It also accepts a list of inputs as children, which can be either [`<InputGuesser>`](#inputguesser) elements, or any [input component](https://marmelab.com/react-admin/Inputs.html) available in React Admin, such as [`<TextInput>`](https://marmelab.com/react-admin/TextInput.html), [`<DateInput>`](https://marmelab.com/react-admin/DateInput.html) or [`<ReferenceInput>`](https://marmelab.com/react-admin/ReferenceInput.html) for instance.

```tsx
// BooksEdit.tsx
import { EditGuesser, InputGuesser } from '@api-platform/admin';
import { DateInput, TextInput, required } from 'react-admin';

export const BooksEdit = () => (
  <EditGuesser>
    {/* InputGuesser comes from API Platform Admin */}
    <InputGuesser source="isbn" label="ISBN" />
    <InputGuesser source="title" />
    <InputGuesser source="author" />

    {/* DateInput and TextInput come from React Admin */}
    <DateInput source="publicationDate" />
    <TextInput
      source="description"
      multiline
      validate={required()}
    />
  </EditGuesser>
);
```

#### EditGuesser Props

`<EditGuesser>` accepts all props accepted by both React Admin [`<Edit>` component](https://marmelab.com/react-admin/Edit.html) and [`<SimpleForm>` component](https://marmelab.com/react-admin/SimpleForm.html).

For instance you can pass props such as [`redirect`](https://marmelab.com/react-admin/Edit.html#redirect), [`mutationMode`](https://marmelab.com/react-admin/Edit.html#mutationmode), [`defaultValues`](https://marmelab.com/react-admin/SimpleForm.html#defaultvalues) or [`warnWhenUnsavedChanges`](https://marmelab.com/react-admin/SimpleForm.html#warnwhenunsavedchanges).

### ShowGuesser

Displays a detailed page for one item. Based on React Admin [`<Show>`](https://marmelab.com/react-admin/Show.html) ans [`<SimpleShowLayout>`](https://marmelab.com/react-admin/SimpleShowLayout.html) components.

If you pass no children, it will display fields guessed from the schema.

```tsx
// BooksShow.tsx
import { ShowGuesser } from '@api-platform/admin';

export const BooksShow = () => (
  /* Will display fields guessed from the schema */
  <ShowGuesser />
);
```

It also accepts a list of fields as children, which can be either [`<FieldGuesser>`](#fieldguesser) elements, or any [field component](https://marmelab.com/react-admin/Fields.html) available in React Admin, such as [`<TextField>`](https://marmelab.com/react-admin/TextField.html), [`<DateField>`](https://marmelab.com/react-admin/DateField.html) or [`<ReferenceField>`](https://marmelab.com/react-admin/ReferenceField.html) for instance.

```tsx
// BooksShow.tsx
import { FieldGuesser, ShowGuesser } from '@api-platform/admin';
import { DateField, NumberField } from 'react-admin';

export const BooksShow = () => (
  <ShowGuesser>
    {/* FieldGuesser comes from API Platform Admin */}
    <FieldGuesser source="isbn" label="ISBN" />
    <FieldGuesser source="title" />
    <FieldGuesser source="author" />

    {/* DateField and NumberField come from React Admin */}
    <DateField source="publicationDate" />
    <NumberField source="reviews.length" label="Reviews" />
  </ShowGuesser>
);
```

#### ShowGuesser Props

`<ShowGuesser>` accepts all props accepted by both React Admin [`<Show>` component](https://marmelab.com/react-admin/Show.html) and [`<SimpleShowLayout>` component](https://marmelab.com/react-admin/SimpleShowLayout.html).

## Hydra

### HydraAdmin

Creates a complete Admin, using [`<AdminGuesser>`](#adminguesser), but configured specially for [Hydra](https://www.hydra-cg.com/).

**Tip:** If you want to use other formats (see supported formats: `@api-platform/api-doc-parser`) use [`<AdminGuesser>`](#adminguesser) instead.

```tsx
// App.tsx
import { HydraAdmin, ResourceGuesser } from '@api-platform/admin';

const App = () => (
  <HydraAdmin entrypoint="https://demo.api-platform.com">
    <ResourceGuesser name="books" />
    {/* ... */}
  </HydraAdmin>
);

export default App;
```

#### HydraAdmin Props

| Name         | Type                | Value        | required | Description                  |
| ------------ | ------------------- | ------------ | -------- | ---------------------------- |
| entrypoint   | string              | -            | yes      | entrypoint of the API        |
| mercure      | object&#124;boolean | \*           | no       | configuration to use Mercure |
| dataProvider | object              | dataProvider | no       | hydra data provider to use   |

\* `false` to explicitly disable, `true` to enable with default parameters or an object with the following properties:

- `hub`: the URL to your Mercure hub
- `jwt`: a subscriber JWT to access your Mercure hub
- `topicUrl`: the topic URL of your resources

### Hydra Data Provider

An implementation for the React Admin [dataProvider methods](https://marmelab.com/react-admin/DataProviderWriting.html): `create`, `delete`, `getList`, `getManyReference`, `getOne` and `update`.

The `dataProvider` is used by API Platform Admin to communicate with the API.

In addition, the specific `introspect` method parses your API documentation.

Note that the `dataProvider` can be overridden to fit your API needs.

### Hydra Schema Analyzer

Analyses your resources and retrieves their types according to the [Schema.org](https://schema.org) vocabulary.

## OpenAPI

### OpenApiAdmin

Creates a complete Admin, as [`<AdminGuesser>`](#adminguesser), but configured specially for [OpenAPI](https://www.openapis.org/).

**Tip:** If you want to use other formats (see supported formats: `@api-platform/api-doc-parser`) use [`<AdminGuesser>`](#adminguesser) instead.

```tsx
// App.tsx
import { OpenApiAdmin, ResourceGuesser } from '@api-platform/admin';

const App = () => (
  <OpenApiAdmin
    entrypoint={entrypoint}
    docEntrypoint={docEntrypoint}
  >
    <ResourceGuesser name="books" />
    {/* ... */}
  </OpenApiAdmin>
);

export default App;
```

#### OpenApiAdmin Props

| Name          | Type                | Value | required | Description                  |
| ------------- | ------------------- | ----- | -------- | ---------------------------- |
| docEntrypoint | string              | -     | yes      | doc entrypoint of the API    |
| entrypoint    | string              | -     | yes      | entrypoint of the API        |
| dataProvider  | dataProvider        | -     | no       | data provider to use         |
| mercure       | object&#124;boolean | \*    | no       | configuration to use Mercure |

\* `false` to explicitly disable, `true` to enable with default parameters or an object with the following properties:

- `hub`: the URL to your Mercure hub
- `jwt`: a subscriber JWT to access your Mercure hub
- `topicUrl`: the topic URL of your resources

### Open API Data Provider

An implementation for the React Admin [dataProvider methods](https://marmelab.com/react-admin/DataProviderWriting.html): `create`, `delete`, `getList`, `getManyReference`, `getOne` and `update`.

The `dataProvider` is used by API Platform Admin to communicate with the API.

In addition, the specific `introspect` method parses your API documentation.

Note that the `dataProvider` can be overridden to fit your API needs.

### Open API Schema Analyzer

Analyses your resources and retrieves their types according to the [Schema.org](https://schema.org) vocabulary.

## Other Components

### FieldGuesser

Renders a field according to its type, using the [schema analyzer](#hydra-schema-analyzer).

Based on React Admin [field components](https://marmelab.com/react-admin/Fields.html), such as [`<TextField>`](https://marmelab.com/react-admin/TextField.html), [`<DateField>`](https://marmelab.com/react-admin/DateField.html) or [`<ReferenceField>`](https://marmelab.com/react-admin/ReferenceField.html).

```tsx
// BooksShow.tsx
import { FieldGuesser, ShowGuesser } from '@api-platform/admin';

export const BooksShow = () => (
  <ShowGuesser>
    {/* Renders a TextField */}
    <FieldGuesser source="title" />
    {/* Renders a NumberField */}
    <FieldGuesser source="rating" />
    {/* Renders a DateField */}
    <FieldGuesser source="publicationDate" />
  </ShowGuesser>
);
```

#### FieldGuesser Props

| Name   | Type   | Value | required | Description                          |
| ------ | ------ | ----- | -------- | ------------------------------------ |
| source | string | -     | yes      | name of the property of the resource |

`<FieldGuesser>` also accepts any [common field prop](https://marmelab.com/react-admin/Fields.html#common-field-props) supported by React Admin, such as [`label`](https://marmelab.com/react-admin/Fields.html#label) for instance.

### InputGuesser

Renders an input according to its type, using the [schema analyzer](#hydra-schema-analyzer).

Uses React Admin [input components](https://marmelab.com/react-admin/Inputs.html), such as [`<TextInput>`](https://marmelab.com/react-admin/TextInput.html), [`<DateInput>`](https://marmelab.com/react-admin/DateInput.html) or [`<ReferenceInput>`](https://marmelab.com/react-admin/ReferenceInput.html).

```tsx
// BooksCreate.tsx
import { CreateGuesser, InputGuesser } from '@api-platform/admin';

export const BooksCreate = () => (
  <CreateGuesser>
    {/* Renders a TextInput */}
    <InputGuesser source="title" />
    {/* Renders a NumberInput */}
    <InputGuesser source="rating" />
    {/* Renders a DateInput */}
    <InputGuesser source="publicationDate" />
  </CreateGuesser>
);
```

#### InputGuesser Props

| Name   | Type   | Value | required | Description                          |
| ------ | ------ | ----- | -------- | ------------------------------------ |
| source | string | -     | yes      | name of the property of the resource |

`<InputGuesser>` also accepts any [common input prop](https://marmelab.com/react-admin/Inputs.html#common-input-props) supported by React Admin, such as [`defaultValue`](https://marmelab.com/react-admin/Inputs.html#defaultvalue), [`readOnly`](https://marmelab.com/react-admin/Inputs.html#readonly), [`helperText`](https://marmelab.com/react-admin/Inputs.html#helpertext) or [`label`](https://marmelab.com/react-admin/Inputs.html#label).

You can also pass props that are specific to a certain input component. For example, if you know an `<InputGuesser>` will render a `<TextInput>` and you would like that input to be multiline, you can set the [`multiline`](https://marmelab.com/react-admin/TextInput.html#multiline) prop.

```tsx
<InputGuesser source="description" multiline />
```

