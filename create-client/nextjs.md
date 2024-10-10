# Next.js Generator

![List screenshot](images/nextjs/create-client-nextjs-list.png)

The Next.js generator scaffolds components for server-side rendered (SSR) applications using [Next.js](https://nextjs.org/).

## Install

The easiest way to get started is to install [the API Platform Symfony variant](../symfony/index.md).
It contains a Next.js skeleton generated with Create Next App,
a development Docker container to serve the webapp, and all the API Platform components you may need, including an API server
supporting Hydra and OpenAPI.

If you use API Platform, jump to the next section!

Alternatively, create a Next.js application by executing:

```console
# using pnpm (recommended)
pnpm create next-app --typescript
# or using npm
npm init next-app --typescript
# or using yarn
yarn create next-app --typescript
```

Install the required dependencies:

```console
# using pnpm
pnpm install isomorphic-unfetch formik react-query
# or using npm
npm install isomorphic-unfetch formik react-query
# or using yarn
yarn add isomorphic-unfetch formik react-query
```

The generated HTML will contain [Tailwind CSS](https://tailwindcss.com) classes.
Optionally, [follow the Tailwind installation guide for Next.js projects](https://tailwindcss.com/docs/guides/nextjs)
(Tailwind is preinstalled in [the API Platform Symfony variant](../symfony/index.md))

## Generating Routes

If you use the API Platform symfony variant, generating all the code you need for a given resource is as simple as running the following command:

```console
docker compose exec pwa \
    pnpm create @api-platform/client --resource book -g next
```

Omit the resource flag to generate files for all resource types exposed by the API.

If you don't use the standalone installation, run the following command instead:

```console
# using pnpm
pnpm create @api-platform/client https://demo.api-platform.com . --generator next --resource book
# or using npm
npm init @api-platform/client https://demo.api-platform.com . -- --generator next --resource book
# or using yarn
yarn create @api-platform/client https://demo.api-platform.com . --generator next --resource book
```

Replace the URL by the entrypoint of your Hydra-enabled API.
You can also use an OpenAPI documentation with `-f openapi3`.

The code has been generated, and is ready to be executed!

Add the layout to the app:

```typescript
import type { AppProps } from "next/app";
import type { DehydratedState } from "react-query";

import Layout from "../components/common/Layout";

const App = ({ Component, pageProps }: AppProps<{dehydratedState: DehydratedState}>) => (
  <Layout dehydratedState={pageProps.dehydratedState}>
    <Component {...pageProps} />
  </Layout>
);

export default App;
```

## Starting the Project

You can launch the server with

```console
# using pnpm
pnpm dev
# or using npm
npm run dev
# or using yarn
yarn dev
```

Go to `http://localhost:3000/books/` to start using your app.

## Generating a production build locally with docker compose

If you want to generate a production build locally with docker compose, follow [these instructions](../deployment/docker-compose.md)

## Screenshots

![List](images/nextjs/create-client-nextjs-list.png)
![Show](images/nextjs/create-client-nextjs-show.png)
