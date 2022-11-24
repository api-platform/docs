# Quasar Framework Generator

Create a Quasar Framework application using
[Quasar CLI](https://quasar.dev/start/quasar-cli):

When asked please choose the following options :
- App with Quasar CLI
- Project folder : my-app
- Quasar v2
- Typescript
- Quasar app CLI with Vite
- Composition API with \<script setup\>
- State Management (Pinia) / Vue-i18n

```console
npm i -g @quasar/cli
npm init quasar
cd my-app
```

Install the required dependencies:


```console
npm install dayjs qs
```

In the app directory, generate the files for the resource you want:

```console
npm init @api-platform/client https://demo.api-platform.com src/ -- --generator quasar --resource foo
```

Replace the URL by the entrypoint of your Hydra-enabled API.
You can also use an OpenAPI documentation with `https://demo.api-platform.com/docs.json` and `-f openapi3`.

Omit the resource flag to generate files for all resource types exposed by the API.

The code is ready to be executed! Import common translations:

```javascript
// src/i18n/en-US/index.ts
import common from './common';

export default {
  // ...
  ...common,
}
```

Finally, make sure to update the config:

```javascript
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
