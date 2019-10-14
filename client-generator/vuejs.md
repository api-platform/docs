# Vue.js Generator

Bootstrap a Vue.js application using [vue-cli](https://github.com/vuejs/vue-cli):

    $ vue init webpack-simple my-app
    $ cd my-app

Install the required dependencies:

    $ yarn add vue-router vuex vuex-map-fields babel-plugin-transform-builtin-extend babel-preset-es2015 babel-preset-stage-2 lodash

Optionally, install Bootstrap and Font Awesome to get an app that looks good:

    $ yarn add bootstrap font-awesome

To generate all the code you need for a given resource run the following command:

    $ npx @api-platform/client-generator https://demo.api-platform.com src/ --resource book
    # Replace the URL by the entrypoint of your Hydra-enabled API
    # Omit the resource flag to generate files for all resource types exposed by the API

The code is ready to be executed! Register the generated routes and store modules in the `main.js` file. Here is an example:

```javascript
import Vue from 'vue'
import Vuex from 'vuex';
import VueRouter from 'vue-router';
import App from './App.vue';
import 'bootstrap/dist/css/bootstrap.css';
import 'font-awesome/css/font-awesome.css';
import book from './store/modules/book/';
import bookRoutes from './router/book';

Vue.config.productionTip = false;

Vue.use(Vuex);
Vue.use(VueRouter);

const store = new Vuex.Store({
  modules: {
    book,
  }
});

const router = new VueRouter({
  mode: 'history',
  routes: [
      ...bookRoutes,
  ],
});

new Vue({
  store,
  router,
  render: h => h(App),
}).$mount('#app');

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

Go to `https://localhost/books/` to start using your app.
That's all!
