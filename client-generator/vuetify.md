# Vuetify Generator

## Install with Docker

If you use the API Platform distribution with docker, first you have to add the [Vue CLI](https://cli.vuejs.org/guide/) to the `yarn global add` command in `client/Dockerfile`:

```dockerfile
RUN yarn global add @api-platform/client-generator @vue/cli @vue/cli-service-global
```

Rebuild your containers to install the Vue CLI in docker client container.

Remove the directories `client\src\` and `client\public\` and the files `client\package.json\` and `client\yarn.lock\`  (because the distribution comes with a prebuilt react app.)

Create a new Vue App and install vuetify and other vue packages:

```shell script
docker-compose exec client vue create -d .
docker-compose exec client vue add vuetify
docker-compose exec client yarn add router lodash moment vue-i18n vue-router vuelidate vuex vuex-map-fields
```

Update the entrypoint in the `client/src/config/entrypoint.js` file:

```ecmascript 6
export const ENTRYPOINT = 'https://localhost:8443';
```

Update the scripts part of the new `composer.json`:

```ecmascript 6
  "scripts": {
    "build": "vue-cli-service build",
    "lint": "vue-cli-service lint",
    "start": "vue-cli-service serve --port 3000 --https"
  },
```

Rebuild the docker containers again to install the Vue App and start the vue server.

Generate the vuetify components with the client generator:

```shell script
$ docker-compose exec client generate-api-platform-client -g vuetify --resource book
# Omit the resource flag to generate files for all resource types exposed by the API
```

Continue by [generating the VueJS Web App](#generating-the-vuejs-web-app)

## Install without Docker

Create a Vuetify application using
[Vue CLI 3](https://cli.vuejs.org/guide/):

    $ vue create -d vuetify-app
    $ cd vuetify-app
    $ vue add vuetify

Install the required dependencies:

    $ yarn add router lodash moment vue-i18n vue-router vuelidate vuex vuex-map-fields

    $ yarn global add @api-platform/client-generator

In the app directory, generate the files for the resource you want:

    $ npx @api-platform/client-generator -g vuetify https://demo.api-platform.com src/
    # Replace the URL with the entrypoint of your Hydra-enabled API
    # Omit the resource flag to generate files for all resource types exposed by the API

## Generating the VueJS Web App

The code is ready to be executed! Register the generated routes in `src/router/index.js`

```javascript
import Vue from 'vue';
import VueRouter from 'vue-router';
import bookRoutes from './book';
import reviewRoutes from './review';

Vue.use(VueRouter);

export default new VueRouter({
  mode: 'history',
  routes: [bookRoutes, reviewRoutes]
});

```

and store modules in the `src/store/index.js` file.

```javascript
import Vue from 'vue';
import Vuex from 'vuex';
import makeCrudModule from './modules/crud';
import notifications from './modules/notifications';
import bookService from '../services/book';
import reviewService from '../services/review';

Vue.use(Vuex);

const store = new Vuex.Store({
  modules: {
    notifications,
    book: makeCrudModule({
      service: bookService
    }),
    review: makeCrudModule({
      service: reviewService
    })
  },
  strict: process.env.NODE_ENV !== 'production'
});

export default store;

```

Update the `src/plugins/vuetify.js` file with the following:

```javascript
import Vue from 'vue';
import Vuetify from 'vuetify/lib';

Vue.use(Vuetify);

const opts = {
  icons: {
    iconfont: 'mdi'
  }
};

export default new Vuetify(opts);
```

The generator comes with a i18n feature to allow quick translations of some labels in the generated code, to make it
work, you need to create the `src/i18n.js` file with the following:

```javascript
import Vue from 'vue';
import VueI18n from 'vue-i18n';
import messages from './locales/en';

Vue.use(VueI18n);

export default new VueI18n({
  locale: process.env.VUE_APP_I18N_LOCALE || 'en',
  fallbackLocale: process.env.VUE_APP_I18N_FALLBACK_LOCALE || 'en',
  messages: {
    en: messages
  }
});
```

Update your App.vue with following:

```javascript
<template>
  <v-app id="inspire">
    <snackbar></snackbar>
    <v-navigation-drawer v-model="drawer" app>
      <v-list dense>
        <v-list-item>
          <v-list-item-action>
            <v-icon>mdi-home</v-icon>
          </v-list-item-action>
          <v-list-item-content>
            <v-list-item-title>Home</v-list-item-title>
          </v-list-item-content>
        </v-list-item>
        <v-list-item>
          <v-list-item-action>
            <v-icon>mdi-book</v-icon>
          </v-list-item-action>
          <v-list-item-content>
            <v-list-item-title>
              <router-link :to="{ name: 'BookList' }">Books</router-link>
            </v-list-item-title>
          </v-list-item-content>
        </v-list-item>
        <v-list-item>
          <v-list-item-action>
            <v-icon>mdi-comment-quote</v-icon>
          </v-list-item-action>
          <v-list-item-content>
            <v-list-item-title>
              <router-link :to="{ name: 'ReviewList' }">Reviews</router-link>
            </v-list-item-title>
          </v-list-item-content>
        </v-list-item>
      </v-list>
    </v-navigation-drawer>
    <v-app-bar app color="indigo" dark>
      <v-app-bar-nav-icon @click.stop="drawer = !drawer"></v-app-bar-nav-icon>
      <v-toolbar-title>Application</v-toolbar-title>
    </v-app-bar>

    <v-content>
      <Breadcrumb layout-class="pl-3 py-3" />
      <router-view></router-view>
    </v-content>
    <v-footer color="indigo" app>
      <span class="white--text">&copy; 2019</span>
    </v-footer>
  </v-app>
</template>

<script>
import Breadcrumb from './components/Breadcrumb';
import Snackbar from './components/Snackbar';

export default {
  components: {
    Breadcrumb,
    Snackbar
  },

  data: () => ({
    drawer: null
  })
};
</script>
```

To finish, update your `main.js` with the following :

```javascript
import Vue from 'vue';
import Vuelidate from 'vuelidate';
import App from './App.vue';
import vuetify from './plugins/vuetify';

import store from './store';
import router from './router';
import i18n from './i18n';

Vue.config.productionTip = false;

Vue.use(Vuelidate);

new Vue({
  i18n,
  router,
  store,
  vuetify,
  render: h => h(App)
}).$mount('#app');

```

Go to `https://localhost:8000/books/` to start using your app.
That's all!
