# Quasar Framework Generator

Create a Quasar Framework application using
[quasar-cli](https://quasar.dev/quasar-cli/installation):

    $ quasar create my-app
    $ cd my-app

    $ yarn global add @api-platform/client-generator

In the app directory, generate the files for the resource you want:

    $ generate-api-platform-client -g quasar https://demo.api-platform.com src/ --resource foo
    # Replace the URL by the entrypoint of your Hydra-enabled API
    # Omit the resource flag to generate files for all resource types exposed by the API

The code is ready to be executed! Register the generated routes in `src/router/routes.js`

```javascript
import fooRoutes from '../generated/router/foo';

const routes = [
  {
    path: '/',
    component: () => import('layouts/MyLayout.vue'),
    children: [
      ...fooRoutes
    ],

```

and store modules in the `src/store/index.js` file.

```javascript
// Replace "foo" with the name of the resource type
import foo from '../generated/store/modules/foo/';

export default function(/* { ssrContext } */) {
  const Store = new Vuex.Store({
    modules: {
      foo,
    },

```

Make sure to update the `quasar.conf.js` file with the following:

```javascript
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

    ...
  ],
  directives: [..., 'ClosePopup'],
  plugins: ['Notify'],
  config: {
    ...
    notify: {
      position: 'top',
      multiLine: true,
      timeout: 0,
    },
  },
```
