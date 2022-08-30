# Next.js Generator

![List screenshot](images/nextjs/client-generator-nextjs-list.png)

The Next.js Client Generator generates components for Server Side Rendered applications using [Next.js](https://nextjs.org/).

## Install

The easiest way to get started is to install [the API Platform distribution](../distribution/index.md).
It contains the Client Generator, all dependencies it needs, a Next.js skeleton generated with Create Next App,
a development Docker container to serve the webapp, and all the API Platform components you may need, including an API server
supporting Hydra.

If you use API Platform, jump to the next section!

Alternatively, create a Next.js application by executing:

```console
npx create-next-app --typescript your-app-name
# or
yarn create next-app --typescript your-app-name
```

Install the required dependencies:

```console
yarn add isomorphic-unfetch formik react-query
```

## Generating Routes

If you use the API Platform distribution, generating all the code you need for a given resource is as simple as running the following command:

```console
docker compose exec client \
    generate-api-platform-client --resource book -g next
```

Omit the resource flag to generate files for all resource types exposed by the API.

If you don't use the standalone installation, run the following command instead:

```console
npx @api-platform/client-generator https://demo.api-platform.com . --generator next --resource book
# Replace the URL by the entrypoint of your Hydra-enabled API.
# You can also use an OpenAPI documentation with `-f openapi3`.
```

The code has been generated, and is ready to be executed!

## Starting the Project

You can launch the server with

```console
yarn dev
```

Go to `http://localhost:3000/books/` to start using your app.

## Screenshots

![List](images/nextjs/client-generator-nextjs-list.png)
![Show](images/nextjs/client-generator-nextjs-show.png)
