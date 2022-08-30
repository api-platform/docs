# React Generator

![List screenshot](images/react/client-generator-react-list.png)

The React Client Generator generates a Single Page Application or a Progressive Web App built with battle-tested libraries
from the ecosystem:

* [React](https://reactjs.org/)
* [React Router](https://reactrouter.com/)
* [React Hook Form](https://react-hook-form.com/)

It is designed to generate code that works seamlessly with [Facebook's Create React App](https://create-react-app.dev/).

## Install

Bootstrap a React application:

```console
npx create-react-app --template typescript client
cd client
```

Install the required dependencies:

```console
yarn add react-router-dom react-hook-form
```

Optionally, install Bootstrap and Font Awesome to get an app that looks good:

```console
yarn add bootstrap font-awesome
```

Finally, start the integrated web server:

```console
yarn start
```

## Generating a Web App

```console
npx @api-platform/client-generator https://demo.api-platform.com src/ --generator next --resource book
# Replace the URL by the entrypoint of your Hydra-enabled API
# Omit the resource flag to generate files for all resource types exposed by the API.
# You can also use an OpenAPI documentation with `-f openapi3`.
```

> Note: On the [API Platform distribution](https://github.com/api-platform/api-platform), you can run
> `generate-api-platform-client` instead of `npx @api-platform/client-generator`.

The code has been generated, and is ready to be executed!

Register the reducers and the routes:

```typescript
// client/src/index.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { BrowserRouter as Router, Route, Routes } from 'react-router-dom';
import 'bootstrap/dist/css/bootstrap.css';
import 'font-awesome/css/font-awesome.css';
// Import your routes here
import App from './App';

const NotFound = () => (
  <h1>Not Found</h1>
);

const root = ReactDOM.createRoot(
  document.getElementById('root') as HTMLElement
);
root.render(
  <React.StrictMode>
    <Router>
      <Routes>
        <Route path="/" component={Welcome} strict={true} exact={true}/>
        {/* Add your routes here */}
        <Route render={() => <h1>Not Found</h1>} />
      </Routes>
    </Router>
  </React.StrictMode>
);
```

Go to `https://localhost/books/` to start using your app.

## Screenshots

![List](images/react/client-generator-react-list.png)
![Pagination](images/react/client-generator-react-list-pagination.png)
![Show](images/react/client-generator-react-show.png)
![Edit](images/react/client-generator-react-edit.png)
![Delete](images/react/client-generator-react-delete.png)
