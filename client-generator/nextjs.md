# Next.js Generator

![List screenshot](images/nextjs/client-generator-nextjs-list.png)

The Next.js Client Generator generates routes and components for Server Side Rendered applications using [Next.js](https://zeit.co/blog/next)

## Install

### Next + Express Server

Create a [Next.js application with express server](https://github.com/zeit/next.js/tree/canary/examples/custom-server-express). The easiest way is to execute:  

    $ npx create-next-app --example custom-server-express your-app-name
    # or
    $ yarn create next-app --example custom-server-express your-app-name

### Enabling Typescript

Install typescript dependencies  

    $ yarn add @types/next @zeit/next-typescript

Enable Typescript in your Next.js configuration file (`next.config.js`):

```javascript
const withTypescript = require('@zeit/next-typescript');

module.exports = withTypescript();
```

Create a `.babelrc` file to store Babel configuration:
```json
{
  "presets": [
    "next/babel",
    "@zeit/next-typescript/babel"
  ]
}
```

Create a `tsconfig.json` file to store Typescript configuration:
```json
{
  "compilerOptions": {
    "allowJs": true,
    "allowSyntheticDefaultImports": true,
    "jsx": "preserve",
    "lib": ["dom", "es2017"],
    "module": "esnext",
    "moduleResolution": "node",
    "noEmit": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "preserveConstEnums": true,
    "removeComments": false,
    "skipLibCheck": true,
    "sourceMap": true,
    "strict": true,
    "target": "esnext",
    "typeRoots": [
      "node_modules/@types"
    ]
  }
}
```

### Installing the Generator Dependencies

Install required dependencies:

    $ yarn add lodash.get lodash.has @types/lodash isomorphic-unfetch

## Starting the Project

You can launch the server with 

    $ yarn dev

and access it through `http://localhost:3000`

## Generating Routes

    $ npx @api-platform/client-generator https://demo.api-platform.com src/ --generator next --resource book
    # Replace the URL by the entrypoint of your Hydra-enabled API

> Note: Omit the resource flag to generate files for all resource types exposed by the API.

If your express server is compatible with the `custom-server-express` Next.js example, you can use the `server-path` flag to specify path to the server file. Routes will be added automatically to this file, otherwise, you will receive some hints on how to them to your own custom server.

    $ npx @api-platform/client-generator https://demo.api-platform.com src/ --generator next --server-path ./server.js

Go to `https://localhost:3000/books/` to start using your app.
That's it!

## Screenshots

![List](images/nextjs/client-generator-nextjs-list.png)  
![Show](images/nextjs/client-generator-nextjs-show.png)
