# Vue.js Generator

Bootstrap a Vue.js application using create-vue:

```console
npm init vue@3 -- --typescript --router --pinia --eslint-with-prettier my-app
cd my-app
```

Install the required dependencies:

```console
npm install lodash @types/lodash dayjs
```

Optionally, install Bootstrap to get an app that looks good:

```console
npm install bootstrap bootstrap-icons
```

To generate all the code you need for a given resource run the following command:

```console
npm init @api-platform/client https://demo.api-platform.com src/ -- --generator vue --resource book
```

Replace the URL with the entrypoint of your Hydra-enabled API.
You can also use an OpenAPI documentation with `https://demo.api-platform.com/docs.json` and `-f openapi3`.

Omit the resource flag to generate files for all resource types exposed by the API.

The code is ready to be executed! Replace the content of main.ts and App.vue with the following code:

```typescript
// src/main.ts
import { createApp } from "vue";
import { createPinia } from "pinia";

import "bootstrap/dist/css/bootstrap.min.css";
import "bootstrap";
import "bootstrap-icons/font/bootstrap-icons.css";

import App from "./App.vue";
import router from "./router";

const app = createApp(App);

app.use(createPinia());
app.use(router);

app.mount("#app");
```

```html
// src/App.vue
<script setup lang="ts">
import { RouterView } from "vue-router";
</script>

<template>
  <RouterView />
</template>
```

Go to `https://localhost/books/` to start using your app.

In order to Mercure to work you have to use the port 3000.
