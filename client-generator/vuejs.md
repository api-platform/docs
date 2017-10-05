# Vue.js generator

Create a Vue.js application using [vue-cli](https://github.com/vuejs/vue-cli):

    $ vue init webpack-simple my-app
    $ cd my-app

Install Vue Router, Vuex and babel-plugin-transform-builtin-extend (to allow extending bultin types like Error and Array):

    $ yarn add vue-router vuex babel-plugin-transform-builtin-extend babel-preset-es2015 babel-preset-stage-2 

Install the generator globally:

    $ yarn global add @api-platform/client-generator

Reference the Bootstrap CSS stylesheet in `index.html` (optional):

```html
  <!-- ... -->
    <title>vue-generator</title>

    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous">
  </head>
  <!-- ... -->
```

In the app directory, generate the files for the resource you want:

    $ generate-api-platform-client --generator vue https://demo.api-platform.com src/ --resource foo
    # Replace the URL by the entrypoint of your Hydra-enabled API
    # Omit the resource flag to generate files for all resource types exposed by the API

The code is ready to be executed! Register the generated routes and store modules in the `main.js` file, here is an example:

```javascript
import Vue from 'vue'
import Vuex from 'vuex';
import VueRouter from 'vue-router';
import App from './App.vue';

// Replace "foo" by the name of the resource type
import foo from './store/modules/foo/';
import fooRoutes from './routes/foo';

Vue.use(Vuex);
Vue.use(VueRouter);

const store = new Vuex.Store({
  modules: {
    foo
  }
});

const router = new VueRouter({
  mode: 'history',
  routes: [
      ...fooRoutes
  ]
});

new Vue({
  el: '#app',
  store,
  router,
  render: h => h(App)
});
```

Make sure to update the `.babelrc` file with the following:

```javascript
{
  "presets": [
    ["es2015", { "modules": false }],
    ["stage-2"]
  ],
  "plugins": [
    ["babel-plugin-transform-builtin-extend", {
      globals: ["Error", "Array"]
    }]
  ]
}
```

Replace the `App.vue` file with the following :

```html
<template>
  <div class="container">
    <nav>
      <ul class="nav nav-justified">
        <!-- Replace Foo by your resource name -->
        <li class="active"><router-link :to="{ name: 'FooList' }" class="active">Foos</router-link></li>
      </ul>
    </nav>

    <div>
      <router-view></router-view>
    </div>
  </div>
</template>
```
Previous chapter: [React generator](react.md)

Next chapter: [Troubleshooting](troubleshooting.md)
