# Vue.js Generator

Bootstrap a Vue 3 application using create-vue:

```console
npm init vue@latest -- --typescript --router --pinia --eslint-with-prettier my-app
cd my-app
```

Install the required dependencies:

```console
npm install dayjs qs @types/qs
```

To generate all the code you need for a given resource run the following command:

```console
npm init @api-platform/client https://demo.api-platform.com src/ -- --generator vue --resource book
```

Replace the URL with the entrypoint of your Hydra-enabled API.
You can also use an OpenAPI documentation with `https://demo.api-platform.com/docs.jsonopenapi` and `-f openapi3`.

Omit the resource flag to generate files for all resource types exposed by the API.

**Note:** Make sure to follow the result indications of the command to register the routes.

Replace the content of `App.vue` with the following code:

```html
// src/App.vue
<template>
  <RouterView />
</template>

<script setup lang="ts">
  import { RouterView } from 'vue-router';
</script>
```

Optionally, install Tailwind to get an app that looks good:

```console
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

Replace the content of `tailwind.config.js` by:

```js
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ['./index.html', './src/**/*.{vue,js,ts,jsx,tsx}'],
  theme: {
    extend: {},
  },
  plugins: [],
};
```

Replace the content of `src/assets/main.css` by:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

You can launch the server with:

```console
npm run dev
```

Go to `http://localhost:5173/books/` to start using your app.

**Note:** In order to Mercure to work with the demo, you have to use the port 3000.
