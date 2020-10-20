# Components

## Guessers

### AdminGuesser

Creates a complete Admin Context and Interface.
The [`<AdminContext>` component](https://marmelab.com/react-admin/Admin.html#unplugging-the-admin-using-admincontext-and-adminui) is set by default with `dataProvider`, `authProvider`,translation and history props.
`<AdminGuesser>` automatically renders an [`<AdminUI>` component](https://marmelab.com/react-admin/Admin.html#unplugging-the-admin-using-admincontext-and-adminui) for resources exposed by a web API documented with Hydra, OpenAPI or any other format supported by `@api-platform/api-doc-parser`. If child components are passed (usually `<ResourceGuesser>` or `<Resource>` components, but it can be any other React component), they are rendered in the given order. If no children are passed, a `<ResourceGuesser>` component is created for each resource type exposed by the API, in the order they are specified in the API documentation.

### CreateGuesser

Displays a creation page for a single resource. Uses React-Admin's [`<Create>`](https://marmelab.com/react-admin/Edit.html) and [`<SimpleForm>`](https://marmelab.com/react-admin/CreateEdit.html#the-simpleform-component) components.

```javascript
const ReviewsCreate = (props) => (
  <CreateGuesser {...props}>
    <InputGuesser source="author" />
    {/* Use react-admin components directly when you want complex inputs. */}
    <ReferenceInput
      source="book"
      reference="books"
      label="Books"
      filterToQuery={(searchText) => ({ title: searchText })}
    >
      <AutocompleteInput optionText="title" />
    </ReferenceInput>

    <InputGuesser source="rating" />

    {/* While deprecated fields are hidden by default, using an explicit InputGuesser component allows to add them back. */}
    <InputGuesser source="letter" />

    <InputGuesser source="body" />
    <InputGuesser source="publicationDate" />
  </CreateGuesser>
);
```

### FieldGuesser

Renders fields according to their types, using hydraSchemaAnalyzer. Based on React-Admin's  [`<ReferenceField>` component](https://marmelab.com/react-admin/Fields.html#referencefield) or[`<ReferenceArrayField>`](https://marmelab.com/react-admin/Fields.html#referencearrayfield) and  [`<SingleFieldList>`](https://marmelab.com/react-admin/List.html#the-singlefieldlist-component) components or for embedded fields on [`<ArrayField>`](https://marmelab.com/react-admin/Fields.html#arrayfield) and [`<SimpleList>`](https://marmelab.com/react-admin/List.html#the-simplelist-component) components.

```javascript
 <FieldGuesser source="author" />
    <FieldGuesser source="book" />
    <ReferenceField label="Book's title" source="book" reference="books">
      <TextField source="title" />
    </ReferenceField>
```

### EditGuesser

Displays an edition page for a single resource. Uses React-Admin's [`<Edit>`](https://marmelab.com/react-admin/Edit.html) and [`<SimpleForm>`](https://marmelab.com/react-admin/CreateEdit.html#the-simpleform-component) components.

```javascript
const ReviewsEdit = (props) => (
  <EditGuesser {...props}>
    <InputGuesser source="author" />

    {/* Use react-admin components directly when you want complex inputs. */}
    <ReferenceInput
      source="book"
      reference="books"
      label="Books"
      filterToQuery={(searchText) => ({ title: searchText })}
    >
      <AutocompleteInput optionText="title" />
    </ReferenceInput>

    <InputGuesser source="rating" />

    {/* While deprecated fields are hidden by default, using an explicit InputGuesser component allows to add them back. */}
    <InputGuesser source="letter" />

    <InputGuesser source="body" />
    <InputGuesser source="publicationDate" />
  </EditGuesser>
);
```

### InputGuesser

Uses React-Admin's [`<ReferenceInput>`](https://marmelab.com/react-admin/Inputs.html#referenceinput) for foreign-key values or [`<ReferenceArrayInput>`](https://marmelab.com/react-admin/Inputs.html#referencearrayinput) for arrays of referenced values.

```javascript
  <ReferenceInput
      source="book"
      reference="books"
      label="Books"
      filterToQuery={(searchText) => ({ title: searchText })}
    >
      <AutocompleteInput optionText="title" />
    </ReferenceInput>

    <InputGuesser source="rating" />
```

### ListGuesser

Based on React-Admin's [`<List>`](https://marmelab.com/react-admin/List.html), ListGuesser displays ressource's list in a [`<Datagrid>`](https://marmelab.com/react-admin/List.html#the-datagrid-component), allowing you to use `hasShow` and `hasEdit` props, in order to display or not show and edit buttons (set to `true` by default). It uses Api Platform Admin's Introspecter component to display the ressource's list according to children passed to it (usually `<FieldGuesser>` or `<InputGuesser>`).

```javascript
const ReviewsList = (props) => (
  <ListGuesser {...props}>
    <FieldGuesser source="author" />
    <FieldGuesser source="book" />
    {/* Use react-admin components directly when you want complex fields. */}
    <ReferenceField label="Book's title" source="book" reference="books">
      <TextField source="title" />
    </ReferenceField>

    {/* While deprecated fields are hidden by default, using an explicit FieldGuesser component allows to add them back. */}
    <FieldGuesser source="letter" />
  </ListGuesser>
);
```

### ResourceGuesser

Based on React-Admin's [`<Resource>` component](https://marmelab.com/react-admin/Resource.html), the ResourceGuesser links to CreateGuesser, ListGuesser, EditGuesser, ShowGuesser, if avaiable. Otherwise you can pass it your own CRUD components using `create`, `list`, `edit`, `show` props.

```javascript
export default () => (
  <HydraAdmin
    entrypoint={entrypoint}
    dataProvider={dataProvider}
    authProvider={authProvider}
    loginPage={Login}
  >
    <ResourceGuesser name="books" />
    <ResourceGuesser
      name="reviews"
      list={ReviewsList}
      show={ReviewsShow}
      create={ReviewsCreate}
      edit={ReviewsEdit}
    />

    {/* While deprecated resources are hidden by default, using an explicit ResourceGuesser component allows to add them back. */}
    <ResourceGuesser name="parchments" />
  </HydraAdmin>
);
```

### ShowGuesser

Based on React-Admin's [`<Show>` component](https://marmelab.com/react-admin/Show.html). The `<ShowGuesser>` displays a detailled page for one resource.

```javascript
const ReviewsShow = (props) => (
  <ShowGuesser {...props}>
    <FieldGuesser source="author" addLabel={true} />
    <FieldGuesser source="book" addLabel={true} />
    <FieldGuesser source="rating" addLabel={true} />

    {/* While deprecated fields are hidden by default, using an explicit FieldGuesser component allows to add them back. */}
    <FieldGuesser source="letter" addLabel={true} />

    <FieldGuesser source="body" addLabel={true} />
    <FieldGuesser source="publicationDate" addLabel={true} />
  </ShowGuesser>
);
```

## Provider

### HydraAdmin

```javascript
export default () => (
  <HydraAdmin
    entrypoint={entrypoint}
    dataProvider={dataProvider}
    authProvider={authProvider}
    loginPage={Login}
  >
    <ResourceGuesser name="books" />
    { /* ... */ }
  </HydraAdmin>
);
```

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
