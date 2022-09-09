# Vue.js Generator

Bootstrap a Vue.js application using create-vue:

```console
npm init vue@2 -- --router my-app
cd my-app
```

Install the required dependencies:

```console
npm install vuex@3 vuex-map-fields lodash
```

Optionally, install Bootstrap and Font Awesome to get an app that looks good:

```console
npm install add bootstrap font-awesome
```

To generate all the code you need for a given resource run the following command:

```console
npm init @api-platform/client https://demo.api-platform.com src/ --generator vue --resource book
```

Replace the URL with the entrypoint of your Hydra-enabled API.
You can also use an OpenAPI documentation with `-f openapi3`.

Omit the resource flag to generate files for all resource types exposed by the API.

The code is ready to be executed! Register the generated routes and store modules. Here is an example:

```javascript
// main.js
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

Go to `https://localhost/books/` to start using your app.
