# Quasar Framework Generator

Create a Quasar Framework application using
[quasar-cli](https://quasar.dev/quasar-cli/installation):

```console
quasar create my-app
cd my-app
```

In the app directory, generate the files for the resource you want:

```console
npx @api-platform/client-generator https://demo.api-platform.com src/ --generator quasar --resource foo
```

Replace the URL by the entrypoint of your Hydra-enabled API.
Omit the resource flag to generate files for all resource types exposed by the API.

The code is ready to be executed! Register the generated routes:

```javascript
// src/router/routes.js
import fooRoutes from '../generated/router/foo';

const routes = [
  {
    path: '/',
    component: () => import('layouts/MyLayout.vue'),
    children: [
      ...fooRoutes
    ],
```

And add the modules to the store:

```javascript
// src/store/index.js
// Replace "foo" with the name of the resource type
import foo from '../generated/store/modules/foo/';

export default function(/* { ssrContext } */) {
  const Store = new Vuex.Store({
    modules: {
      foo,
    },
```

Finally, make sure to update the config:

```javascript
// quasar.conf.js
framework: {
  components: [
    'QTable',
    'QTh',
    'QTr',
    'QTd',
    'QAjaxBar',
    'QBreadcrumbs',
    'QBreadcrumbsEl',
    'QSpace',
    'QInput',
    'QForm',
    'QSelect',
    'QMarkupTable',
    'QDate',
    'QTime',
    'QCheckbox',
    'QPopupProxy'

    // ...
  ],
  directives: [..., 'ClosePopup'],
  plugins: ['Notify'],
  config: {
    // ...
    notify: {
      position: 'top',
      multiLine: true,
      timeout: 0,
    },
  },
```
