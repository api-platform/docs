# Vuetify Generator

Bootstrap a Vuetify 3 application using `create-vuetify`:

```console
npm init vuetify -- --typescript --preset essentials
cd my-app
```

Install the required dependencies:

```console
npm install dayjs qs @types/qs vue-i18n
```

To generate all the code you need for a given resource run the following command:

```console
npm init @api-platform/client https://demo.api-platform.com src/ -- --generator vuetify --resource book
```

Replace the URL with the entrypoint of your Hydra-enabled API.
You can also use an OpenAPI documentation with `https://demo.api-platform.com/docs.jsonopenapi` and `-f openapi3`.

Omit the resource flag to generate files for all resource types exposed by the API.

**Note:** Make sure to follow the result indications of the command to register the routes and the translations.

Then add this import in `src/plugins/vuetify.ts`:

```typescript
// src/plugins/vuetify.ts
import { VDataTableServer } from 'vuetify/labs/VDataTable';
```

In the same file replace the export with:

```typescript
// src/plugins/vuetify.ts
export default createVuetify({
  components: {
    VDataTableServer,
  },
});
```

In `src/plugins/index.ts` add this import:

```typescript
// src/plugins/index.ts
import i18n from '@/plugins/i18n';
```

In the same file add `.use(i18n)` chained with the other `use()` functions.

You can launch the server with:

```console
npm run dev
```

Go to `http://localhost:3000/books/` to start using your app.

**Note:** In order to Mercure to work with the demo, you have to use the port 3000.
