# Vue.js Generator

Create a Vue.js application using [vue-cli](https://github.com/vuejs/vue-cli):

    $ vue init webpack-simple my-app
    $ cd my-app

Install Vue Router, Vuex and babel-plugin-transform-builtin-extend (to allow extending built-in types like Error and Array):

    $ yarn add vue-router vuex babel-plugin-transform-builtin-extend babel-preset-es2015 babel-preset-stage-2

Install the generator globally:

    $ yarn global add @api-platform/client-generator

Reference the Bootstrap CSS stylesheet in `public/index.html` (optional):

```html
  <!-- ... -->
    <title>vue-generator</title>

    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/css/bootstrap.min.css" integrity="sha384-Gn5384xqQ1aoWXA+058RXPxPg6fy4IWvTNh0E263XmFcJlSAwiGgFAW/dAiS6JXm" crossorigin="anonymous">
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/js/bootstrap.min.js" integrity="sha384-JZR6Spejh4U02d8jOt6vLEHfe/JQGiRRSQQxSfFWpi1MquVdAyjUar5+76PVCmYl" crossorigin="anonymous"></script>
  </head>
  <!-- ... -->
```

In the app directory, generate the files for the resource you want:

    $ generate-api-platform-client -g vue https://demo.api-platform.com src/ --resource foo
    # Replace the URL by the entrypoint of your Hydra-enabled API
    # Omit the resource flag to generate files for all resource types exposed by the API

The code is ready to be executed! Register the generated routes and store modules in the `main.js` file, here is an example:

```javascript
import Vue from 'vue'
import Vuex from 'vuex';
import VueRouter from 'vue-router';
import App from './App.vue';

// Replace "foo" with the name of the resource type
import foo from './store/modules/foo/';
import fooRoutes from './router/foo';

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
        <!-- Replace Foo with your resource name -->
        <li class="active"><router-link :to="{ name: 'FooList' }" class="active">Foos</router-link></li>
      </ul>
    </nav>

    <div>
      <router-view></router-view>
    </div>
  </div>
</template>
```
