# Components

## Resource Components

### AdminGuesser

`<AdminGuesser>` renders automatically an [Admin component](https://marmelab.com/react-admin/Admin.html) for resources exposed by a web API documented with any format supported by `@api-platform/api-doc-parser` (for Hydra documented APIs,
use the [HydraAdmin component](components.md#hydraadmin) instead).
It also creates a [schema analyzer](components.md#schema-analyzer) context, where the `schemaAnalyzer` service (for getting information about the provided API documentation) is stored.

`<AdminGuesser>` renders all exposed resources by default, but you can choose what resource you want to render by passing [ResourceGuesser components](components.md#resourceguesser) as children.
Deprecated resources are hidden by default, but you can add them back using an explicit `<ResourceGuesser>` component.

```javascript
// App.js
import { AdminGuesser, ResourceGuesser } from "@api-platform/admin";

const App = () => (
  <AdminGuesser
    entrypoint={entrypoint}
    dataProvider={dataProvider}
    authProvider={authProvider}>
    <ResourceGuesser
      name"books"
      list={BooksList}
      show={BooksShow}
      edit={BooksEdit}
      create={BooksCreate} />
    <ResourceGuesser name"authors" />
  </AdminGuesser>
)

export default App;
```

#### Props

| Name              | Type               | Value          | required | Description                                                                      |
|-------------------|--------------------|----------------|----------|----------------------------------------------------------------------------------|
| dataProvider      | object or function | -              | yes      | communicates with your API                                                       |
| schemaAnalyzer    | object             | schemaAnalyzer | yes      | retrieves resource type according to [Schema.org](https://schema.org) vocabulary |
| children          | node or function   | -              | no       | -                                                                                |
| theme             | object             | theme          | no       | theme of your Admin App                                                          |
| includeDeprecated | boolean            | true or false  | no       | displays or not deprecated resources                                             |

### ResourceGuesser

Based on React Admin [Resource component](https://marmelab.com/react-admin/Resource.html), `<ResourceGuesser>` provides default props [CreateGuesser](components.md#createguesser), [ListGuesser](components.md#listguesser), [EditGuesser](components.md#editguesser) and [ShowGuesser](components.md#showguesser).

Otherwise, you can pass it your own CRUD components using `create`, `list`, `edit`, `show` props.

```javascript
// App.js
import { AdminGuesser, ResourceGuesser } from "@api-platform/admin";

const App = () => (
  <AdminGuesser
    entrypoint={entrypoint}
    dataProvider={dataProvider}
    schemaAnalyzer={schemaAnalyzer}
  >
    <ResourceGuesser
      name="books"
      list={BooksList}
      show={BooksShow}
      create={BooksCreate}
      edit={BooksEdit} />
    <ResourceGuesser name="reviews" />
  </AdminGuesser>
);

export default App;
```

#### ResourceGuesser Props

| Name | Type   | Value | required | Description              |
|------|--------|-------|----------|--------------------------|
| name | string | -     | yes      | endpoint of the resource |

You can also use props accepted by React Admin [Resource component](https://marmelab.com/react-admin/Resource.html). For example, the props `list`, `show`, `create` or `edit`.

## Page Components

### ListGuesser

Based on React Admin [List](https://marmelab.com/react-admin/List.html), `<ListGuesser>` displays a list of resources in a [Datagrid](https://marmelab.com/react-admin/List.html#the-datagrid-component), according to children passed to it (usually [FieldGuesser](components.md#fieldguesser) or any [field component](https://marmelab.com/react-admin/Fields.html#basic-fields)
available in React Admin).

Use `hasShow` and `hasEdit` props if you want to display `show` and `edit` buttons (both set to `true` by default).

By default, `<ListGuesser>` comes with [Pagination](components.md#pagination).

```javascript
// BooksList.js
import { FieldGuesser, ListGuesser } from "@api-platform/admin";
import { ReferenceField, TextField } from "react-admin";

export const BooksList = props => (
  <ListGuesser {...props}>
    <FieldGuesser source="author" />
    <FieldGuesser source="title" />
    <FieldGuesser source="rating" />
    <FieldGuesser source="description" />
    <FieldGuesser source="publicationDate" />
  </ListGuesser>
);
```

#### ListGuesser Props

| Name     | Type             | Value | required | Description                             |
|----------|------------------|-------|----------|-----------------------------------------|
| children | node or function | -     | no       | -                                       |
| resource | string           | -     | yes      | endpoint of the resource                |
| filters  | element          | -     | no       | filters that can be applied to the list |

You can also use props accepted by React Admin [List](https://marmelab.com/react-admin/List.html).

### CreateGuesser

Displays a creation page for a single item. Uses React Admin [Create](https://marmelab.com/react-admin/CreateEdit.html) and [SimpleForm](https://marmelab.com/react-admin/CreateEdit.html#the-simpleform-component) components.
For simple inputs, you can pass as children API Platform Admin [InputGuesser](components.md#inputguesser), or any React Admin [Input components](https://marmelab.com/react-admin/Inputs.html#input-components) for more complex inputs.

```javascript
// BooksCreate.js
import { CreateGuesser, InputGuesser } from "@api-platform/admin";

export const BooksCreate = props => (
  <CreateGuesser {...props}>
    <InputGuesser source="author" />
    <InputGuesser source="title" />
    <InputGuesser source="rating" />
    <InputGuesser source="description" />
    <InputGuesser source="publicationDate" />
  </CreateGuesser>
);
```

#### CreateGuesser Props

| Name     | Type             | Value | required | Description              |
|----------|------------------|-------|----------|--------------------------|
| children | node or function | -     | no       | -                        |
| resource | string           | -     | yes      | endpoint of the resource |

You can also use props accepted by React Admin [Create](https://marmelab.com/react-admin/CreateEdit.html).

### EditGuesser

Displays an edition page for a single item. Uses React Admin [Edit](https://marmelab.com/react-admin/CreateEdit.html) and [SimpleForm](https://marmelab.com/react-admin/CreateEdit.html#the-simpleform-component) components.
For simple inputs, you can use API Platform Admin [InputGuesser](components.md#inputguesser), or any React Admin [Input components](https://marmelab.com/react-admin/Inputs.html#input-components) for more complex inputs.

```javascript
// BooksEdit.js
import { EditGuesser, InputGuesser } from "@api-platform/admin";

export const BooksEdit = props => (
  <EditGuesser {...props}>
    <InputGuesser source="author" />
    <InputGuesser source="title" />
    <InputGuesser source="rating" />
    <InputGuesser source="description" />
    <InputGuesser source="publicationDate" />
  </EditGuesser>
);
```

#### EditGuesser Props

| Name     | Type             | Value | required | Description              |
|----------|------------------|-------|----------|--------------------------|
| children | node or function | -     | no       | -                        |
| resource | string           | -     | yes      | endpoint of the resource |

You can also use props accepted by React Admin [Edit](https://marmelab.com/react-admin/CreateEdit.html).

### ShowGuesser

Displays a detailed page for one item. Based on React Admin [Show component](https://marmelab.com/react-admin/Show.html). You can pass [FieldGuesser](components.md#fieldguesser) as children for simple fields, or use any of React Admin [basic fields](https://marmelab.com/react-admin/Fields.html#basic-fields) for more complex fields.

```javascript
// BooksShow.js
import { FieldGuesser, ShowGuesser } from "@api-platform/admin";

export const BooksShow = props => (
  <ShowGuesser {...props}>
    <FieldGuesser source="author" />
    <FieldGuesser source="title" />
    <FieldGuesser source="rating" />
    <FieldGuesser source="description" />
    <FieldGuesser source="publicationDate" />
  </ShowGuesser>
);
```

#### ShowGuesser Props

| Name     | Type             | Value | required | Description              |
|----------|------------------|-------|----------|--------------------------|
| children | node or function | -     | no       | -                        |
| resource | string           | -     | yes      | endpoint of the resource |

You can also use props accepted by React Admin [Show component](https://marmelab.com/react-admin/Show.html).

## Hydra

### HydraAdmin

Creates a complete Admin, as [AdminGuesser](components.md#adminguesser), but configured specially for [Hydra](https://www.hydra-cg.com/).
If you want to use other formats (see supported formats: `@api-platform/api-doc-parser`) use [AdminGuesser](components.md#adminguesser) instead.

```javascript
// App.js
import { HydraAdmin, ResourceGuesser } from "@api-platform/admin";

const App = () => (
  <HydraAdmin
    entrypoint={entrypoint}
    dataProvider={dataProvider}
    authProvider={authProvider}
   >
     <ResourceGuesser name="books" />
     { /* ... */ }
  </HydraAdmin>
);

export default App;
```

#### HydraAdmin Props

| Name       | Type           | Value | required | Description                  |
|------------|----------------|-------|----------|------------------------------|
| entrypoint | string         | -     | yes      | entrypoint of the API        |
| mercure    | boolean|object | *     | yes      | configuration to use Mercure |

\* `false` to explicitly disable, `true` to enable with default parameters or an object with the following properties:
- `hub`: the URL to your Mercure hub
- `jwt`: a subscriber JWT to access your Mercure hub
- `topicUrl`: the topic URL of your resources

### Data Provider

Based on React Admin `create`, `delete`, `getList`, `getManyReference`, `getOne`, `update` methods, the `dataProvider` is used by API Platform Admin to communicate with the API.
In addition, the specific `introspect` method parses your API documentation.
Note that the `dataProvider` can be overridden to fit your API needs.

### Schema Analyzer

Analyses your resources and retrieves their types according to the [Schema.org](https://schema.org) vocabulary.

## Other Components

### Pagination

Set by default in the [ListGuesser component](components.md#listguesser), the `Pagination` component uses React Admin [Pagination component](https://marmelab.com/react-admin/List.html#pagination).
By default, it renders 30 items per page and displays a navigation UI.
If you want to change the number of items per page or disable the pagination, see the [Pagination documentation](../core/pagination.md).
It is also capable to handle partial pagination.

### FieldGuesser

Renders fields according to their types, using the [schema analyzer](components.md#schemaanalyzer).
Based on React Admin [field components](https://marmelab.com/react-admin/Fields.html).

```javascript
// BooksShow.js
import { FieldGuesser, ShowGuesser } from "@api-platform/admin";

export const BooksShow = props => (
  <ShowGuesser {...props}>
    <FieldGuesser source="author" />
    <FieldGuesser source="title" />
    <FieldGuesser source="rating" />
    <FieldGuesser source="description" />
    <FieldGuesser source="publicationDate" />
  </ShowGuesser>
)
```

#### FieldGuesser Props

| Name   | Type   | Value | required | Description              |
|--------|--------|-------|----------|--------------------------|
| source | string | -     | yes      | endpoint of the resource |

You can also use props accepted by React Admin [basic fields](https://marmelab.com/react-admin/Fields.html#basic-fields).

### InputGuesser

Uses React Admin [input components](https://marmelab.com/react-admin/Inputs.html) to generate inputs according to your API documentation (e.g. number HTML input for numbers, checkbox for booleans, selectbox for relationships...).

#### InputGuesser Props

| Name   | Type   | Value | required | Description              |
|--------|--------|-------|----------|--------------------------|
| source | string | -     | yes      | endpoint of the resource |
