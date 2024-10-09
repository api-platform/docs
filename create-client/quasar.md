# Quasar Framework Generator

Create a Quasar Framework application using
[Quasar CLI](https://quasar.dev/start/quasar-cli):

```console
npm i -g @quasar/cli
npm init quasar
cd my-app
```

It will ask you some questions, you can use these answers:

```console
What would you like to build ? App with Quasar CLI, let's go!
Project folder: my-app
Pick Quasar version: Quasar v2 (Vue 3 | latest and greatest)
Pick script types: Typescript
Pick Quasar App CLI variant: Quasar App CLI with Vite
Package name: my-app
Pick a Vue component style: Composition API with <script setup>
Check the features needed for your project: ESLint / State Management (Pinia) / Vue-i18n
Pick an ESLint Preset: Prettier
```

Install the required dependencies:

```console
npm install dayjs qs @types/qs
```

In the app directory, generate the files for the resource you want:

```console
npm init @api-platform/client https://demo.api-platform.com src/ -- --generator quasar --resource foo
```

Replace the URL by the entrypoint of your Hydra-enabled API.
You can also use an OpenAPI documentation with `https://demo.api-platform.com/docs.jsonopenapi` and `-f openapi3`.

Omit the resource flag to generate files for all resource types exposed by the API.

**Note:** Make sure to follow the result indications of the command to register the routes and the translations.

Import common translations:

```ts
// src/i18n/en-US/index.ts
import common from './common';

export default {
  // ...
  ...common,
};
```

Finally, make sure to update the config:

```js
// quasar.conf.js
framework: {
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

You can launch the server with:

```console
quasar dev
```

Go to `http://localhost:9000/books/` to start using your app.

**Note:** In order to Mercure to work with the demo, you have to use the port 3000.
