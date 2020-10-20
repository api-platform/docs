# Components

## Guessers

### AdminGuesser

Creates a complete Admin Context and Interface.
The [`<AdminContext>` component](https://marmelab.com/react-admin/Admin.html#unplugging-the-admin-using-admincontext-and-adminui) is set by default with `dataProvider`, `authProvider`,translation and history props.
`<AdminGuesser>` automatically renders an [`<AdminUI>` component](https://marmelab.com/react-admin/Admin.html#unplugging-the-admin-using-admincontext-and-adminui) for resources exposed by a web API documented with Hydra, OpenAPI or any other format supported by `@api-platform/api-doc-parser`. If child components are passed (usually `<ResourceGuesser>` or `<Resource>` components, but it can be any other React component), they are rendered in the given order. If no children are passed, a `<ResourceGuesser>` component is created for each resource type exposed by the API, in the order they are specified in the API documentation.

### CreateGuesser

Displays a creation page for a single resource. Uses React-Admin's [`<Create>`](https://marmelab.com/react-admin/Edit.html) and [`<SimpleForm>`](https://marmelab.com/react-admin/CreateEdit.html#the-simpleform-component) components.

### FieldGuesser

Renders fields according to their types, using hydraSchemaAnalyzer. Based on React-Admin's [`<ReferenceArrayField>`](https://marmelab.com/react-admin/Fields.html#referencearrayfield) and  [`<SingleFieldList>`](https://marmelab.com/react-admin/List.html#the-singlefieldlist-component) components or [`<ArrayField>`](https://marmelab.com/react-admin/Fields.html#arrayfield) and [`<SimpleList>`](https://marmelab.com/react-admin/List.html#the-simplelist-component) components for embedded fields.

### EditGuesser

Displays an edition page for a single resource. Uses React-Admin's [`<Edit>`](https://marmelab.com/react-admin/Edit.html) and [`<SimpleForm>`](https://marmelab.com/react-admin/CreateEdit.html#the-simpleform-component) components.

### InputGuesser

Uses React-Admin's [`<ReferenceInput>`](https://marmelab.com/react-admin/Inputs.html#referenceinput) for foreign-key values or [`<ReferenceArrayInput>`](https://marmelab.com/react-admin/Inputs.html#referencearrayinput) for arrays of referenced values.

### ListGuesser

Based on React-Admin's [`<List>`](https://marmelab.com/react-admin/List.html), ListGuesser displays ressource's list in a [`<Datagrid>`](https://marmelab.com/react-admin/List.html#the-datagrid-component), allowing you to use `hasShow` and `hasEdit` props, in order to display or not show and edit buttons (set to `true` by default). It uses Api Platform Admin's Introspecter component to display the ressource's list according to children passed to it (usually `<FieldGuesser>` or `<InputGuesser>`).

### ResourceGuesser

Based on React-Admin's [`<Resource>` component](https://marmelab.com/react-admin/Resource.html), the ResourceGuesser links to CreateGuesser, ListGuesser, EditGuesser, ShowGuesser, if avaiable. Otherwise you can pass it your own CRUD components using `create`, `list`, `edit`, `show` props.

### ShowGuesser

Based on React-Admin's [`<Show>` component](https://marmelab.com/react-admin/Show.html). The `<ShowGuesser>` displays a detailled page for one resource.

## Provider

### HydraAdmin

### dataProvider

Based on React-Admin's `Create`, `Delete`, `getList`, `getManyReference`, `GetOne`, `Update` methods, the dataProvider is used by Api Platform Admin to communicate with your API. It can be overloaded to fit your API needs.

### schemaAnalyzer

Retrieves resources types based on the [Schema.org](https://schema.org) vocabulary.

### fetchHydra

## Other Components

### Introspecter

Introspecter is in charge of the rendering of components, including loading or error if needed.

### Pagination

Uses React-Admin's [`<Pagination>` component](https://marmelab.com/react-admin/List.html#pagination), to let you choose the number of resources displayed on your `<ListGuesser>` or `<List>` and navigate between pages.
