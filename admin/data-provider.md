# Using the Hydra Data Provider Directly with react-admin

By default, the `HydraAdmin` component shipped with API Platform Admin will generate a convenient admin interface for every resource and every property exposed by the API. But sometimes, you may prefer having full control over the generated admin.

To do so, an alternative approach is [to configure every react-admin component manually](https://marmelab.com/react-admin/Tutorial.html) instead of letting the library generate them, but to still leverage the built-in Hydra [data provider](https://marmelab.com/react-admin/DataProviders.html).

Please note that this approach is **not recommended** and should be only for existing React Admin applications.

```javascript
// admin/src/App.js

import React, { Component } from 'react';
import { Admin, Resource } from 'react-admin';
import parseHydraDocumentation from '@api-platform/api-doc-parser/lib/hydra/parseHydraDocumentation';
import { hydraClient, fetchHydra as baseFetchHydra  } from '@api-platform/admin';
import authProvider from './authProvider';
import { Redirect } from 'react-router-dom';
import Layout from './Component/Layout';
import { UserShow } from './Components/User/Show';
import { UserEdit } from './Components/User/Edit';
import { UserCreate } from './Components/User/Create';
import { UserList } from './Components/User/List';

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
import { Show, SimpleShowLayout, TextField, DateField, EmailField, EditButton } from 'react-admin';

export const UserShow = (props) => (
    <Show { ...props }>
        <SimpleShowLayout>
            <TextField source="originId" label="ID"/>
            <EmailField source="email" label="Email" />
            <TextField source="name" label="Name"/>
            <TextField source="phone" label="Phone"/>
            <DateField source="createdAt" label="Date"/>
            <EditButton />
        </SimpleShowLayout>
    </Show>
);
```
