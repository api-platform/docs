# Components

## Guessers

### AdminGuesser
### CreateGuesser
### FieldGuesser
### EditGuesser
### FilterGuesser
### InputGuesser

Uses React-Admin's [`<ReferenceInput>`](https://marmelab.com/react-admin/Inputs.html#referenceinput) for foreign-key values or [`<ReferenceArrayInput>`](https://marmelab.com/react-admin/Inputs.html#referencearrayinput) for arrays of referenced values.

### ListGuesser

Based on React-Admin's [`<List>`](https://marmelab.com/react-admin/List.html), ListGuesser displays ressource's list in a [`<Datagrid>`](https://marmelab.com/react-admin/List.html#the-datagrid-component), allowing you to use `hasShow` and `hasEdit` props, in order to display or not show and edit buttons (set to `true` by default). It uses Api-Platform Admin's Introspecter component to display the ressource's list according to children passed to it (usually FieldGuesser or InputGuesser).

### ResourceGuesser

Based on React-Admin's [`<Resource>` component](https://marmelab.com/react-admin/Resource.html), the ResourceGuesser links to CreateGuesser, ListGuesser, EditGuesser, ShowGuesser, if avaiable. Otherwise you can pass it your own CRUD components using `create`, `list`, `edit`, `show` props.

### ShowGuesser

Based on React-Admin's [`<Show>` component](https://marmelab.com/react-admin/Show.html). The ShowGuesser displays one resource.

## Introspecter

## Pagination

## Layout

### AppBar
### Layout

## Hydra

### HydraAdmin
### dataProvider
### fetchHydra
