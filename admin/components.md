# Components

## Guessers

### AdminGuesser

**Description**

`<AdminGuesser>` creates a complete Admin Context and Interface. Rendering automatically an [`<AdminUI>` component](https://marmelab.com/react-admin/Admin.html#unplugging-the-admin-using-admincontext-and-adminui) for resources exposed by a web API documented with Hydra, OpenAPI or any other format supported by `@api-platform/api-doc-parser`. (For Hydra, you can use the `<HydraAdmin>` component).

**Props**
| Name | Type | Value | required | Description |
| ---- | ---- | ----- | -------- | ----------- |
| schemaAnalyzer | | | | |
| dataProvider | | | yes | |
| authProvider | | | | |
| i18nProvider | | defaultI18nProvider | | |
| history | | defaultHistory | | |
| customReducers | | | | |
| customSagas | | | | |
| initialState | | | | |
| includeDeprecated | boolean | false | | |
| customRoutes | Array | | | |
| appLayout | | | | deprecated, replaced by `layout` |
| layout | | layout | | |
| loginPage | | | | |
| locale | | | | deprecated |
| theme | object | defaultTheme | | |

**Example**

### CreateGuesser

**Description**

Displays a creation page for a single resource. Uses React-Admin's [`<Create>`](https://marmelab.com/react-admin/Edit.html) and [`<SimpleForm>`](https://marmelab.com/react-admin/CreateEdit.html#the-simpleform-component) components.

**Props**
| Name | Type | Value | required | Description |
| ---- | ---- | ----- | -------- | ----------- |

You can also use props provided by React-Admin's [`<Create>`](https://marmelab.com/react-admin/Edit.html).

**Example**

```javascript
// ReviewsCreate.js

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

**Description**

Renders fields according to their types, using hydraSchemaAnalyzer. Based on React-Admin's  [`<ReferenceField>` component](https://marmelab.com/react-admin/Fields.html#referencefield) or[`<ReferenceArrayField>`](https://marmelab.com/react-admin/Fields.html#referencearrayfield) and  [`<SingleFieldList>`](https://marmelab.com/react-admin/List.html#the-singlefieldlist-component) components or for embedded fields on [`<ArrayField>`](https://marmelab.com/react-admin/Fields.html#arrayfield) and [`<SimpleList>`](https://marmelab.com/react-admin/List.html#the-simplelist-component) components.

**Props**
| Name | Type | Value | required | Description |
| ---- | ---- | ----- | -------- | ----------- |

**Example**

```javascript
 <FieldGuesser source="author" />
    <FieldGuesser source="book" />
    <ReferenceField label="Book's title" source="book" reference="books">
      <TextField source="title" />
    </ReferenceField>
```

### EditGuesser

**Description**
Displays an edition page for a single resource. Uses React-Admin's [`<Edit>`](https://marmelab.com/react-admin/Edit.html) and [`<SimpleForm>`](https://marmelab.com/react-admin/CreateEdit.html#the-simpleform-component) components.

**Props**
| Name | Type | Value | required | Description |
| ---- | ---- | ----- | -------- | ----------- |

You can also use props provided by React-Admin's [`<Edit>`](https://marmelab.com/react-admin/Edit.html).

**Example**

```javascript
// ReviewsEdit.js

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

**Description**

Uses React-Admin's [`<ReferenceInput>`](https://marmelab.com/react-admin/Inputs.html#referenceinput) for foreign-key values or [`<ReferenceArrayInput>`](https://marmelab.com/react-admin/Inputs.html#referencearrayinput) for arrays of referenced values, to generate inputs according to your API documentation (e.g. number HTML input for numbers, checkbox for booleans, selectbox for relationships...)

**Props**
| Name | Type | Value | required | Description |
| ---- | ---- | ----- | -------- | ----------- |


**Example**
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

**Description**

Based on React-Admin's [`<List>`](https://marmelab.com/react-admin/List.html), ListGuesser displays ressource's list in a [`<Datagrid>`](https://marmelab.com/react-admin/List.html#the-datagrid-component), allowing you to use `hasShow` and `hasEdit` props, in order to display or not show and edit buttons (set to `true` by default). It uses Api Platform Admin's Introspecter component to display the ressource's list according to children passed to it (usually `<FieldGuesser>` or `<InputGuesser>`).

**Props**

| Name | Type | Value | required | Description |
| ---- | ---- | ----- | -------- | ----------- |

You can also pass props provided by React-Admin's [`<List>`](https://marmelab.com/react-admin/List.html).

**Example**

```javascript
// ReviewsList.js

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

**Description**

Based on React-Admin's [`<Resource>` component](https://marmelab.com/react-admin/Resource.html), the ResourceGuesser links to CreateGuesser, ListGuesser, EditGuesser, ShowGuesser, if avaiable. Otherwise you can pass it your own CRUD components using `create`, `list`, `edit`, `show` props.

**Props**
| Name | Type | Value | required | Description |
| ---- | ---- | ----- | -------- | ----------- |

You can also use props provided by React-Admin's [`<Resource>` component](https://marmelab.com/react-admin/Resource.html).

**Example**

```javascript
// App.js

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

**Description**
Based on React-Admin's [`<Show>` component](https://marmelab.com/react-admin/Show.html). The `<ShowGuesser>` displays a detailled page for one resource.

**Props**
| Name | Type | Value | required | Description |
| ---- | ---- | ----- | -------- | ----------- |

You can also use props provided by React-Admin's [`<Show>` component](https://marmelab.com/react-admin/Show.html).

**Example**

```javascript
// ReviewsShow.js

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

**Description**

Creates a complete Admin Context and Interface, as `<AdminGuesser>`, but configured specially for Hydra. If you want to use other formats, for example JSON:API, use the `<AdminGuesser>` instead.

**Props**
| Name | Type | Value | required | Description |
| ---- | ---- | ----- | -------- | ----------- |
| dataProvider | object | dataProvider | no | formats the data to fit your API |
| entrypoint | url | ? | yes | the entrypoint of your API |

**Example**

```javascript
// App.js

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

**Description**

Based on React-Admin's `Create`, `Delete`, `getList`, `getManyReference`, `GetOne`, `Update` methods, the dataProvider is used by Api Platform Admin to communicate with your API. It can be overrided to fit your API needs.

**Props**
| Name | Type | Value | required | Description |
| ---- | ---- | ----- | -------- | ----------- |

**Example**

### schemaAnalyzer

**Description**

Analyses your resources and retrieves their types according to the [Schema.org](https://schema.org) vocabulary.

**Props**

| Name | Type | Value | required | Description |
| ---- | ---- | ----- | -------- | ----------- |

**Example**

### fetchHydra

**Description**

Sends HTTP requests to a Hydra API. Uses React-Admin's [`HttpError` class](https://marmelab.com/react-admin/List.html#pagination) to throw and display errors.

**Props**
| Name | Type | Value | required | Description |
| ---- | ---- | ----- | -------- | ----------- |

**Example**

## Other Components

### Introspecter

**Description**

Introspecter is in charge of the rendering of components, including loading or error if needed.

**Props**
| Name | Type | Value | required | Description |
| ---- | ---- | ----- | -------- | ----------- |

**Example**

### Pagination

**Description**

Uses React-Admin's [`<Pagination>` component](https://marmelab.com/react-admin/List.html#pagination). Lets you choose the number of resources displayed on your `<ListGuesser>` or `<List>` and navigate between pages.

**Props**
| Name | Type | Value | required | Description |
| ---- | ---- | ----- | -------- | ----------- |

**Example**
