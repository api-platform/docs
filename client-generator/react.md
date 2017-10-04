# React generator

Create a React application using [Facebook's Create React App](https://github.com/facebookincubator/create-react-app):

    $ create-react-app my-app
    $ cd my-app

Install React Router, Redux, React Redux, React Router Redux, Redux Form and Redux Thunk (to handle AJAX requests):

    $ yarn add redux react-redux redux-thunk redux-form react-router-dom react-router-redux prop-types

Install the generator globally:

    $ yarn global add @api-platform/client-generator

Reference the Bootstrap CSS stylesheet in `public/index.html` (optional):

```html
  <!-- ... -->
    <title>React App</title>

    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous">
  </head>
  <!-- ... -->
```

In the app directory, generate the files for the resource you want:

    $ generate-api-platform-client https://demo.api-platform.com src/ --resource foo
    # Replace the URL by the entrypoint of your Hydra-enabled API
    # Omit the resource flag to generate files for all resource types exposed by the API

The code is ready to be executed! Register the generated reducers and components in the `index.js` file, here is an example:

```javascript
import React from 'react';
import ReactDom from 'react-dom';
import { createStore, combineReducers, applyMiddleware } from 'redux';
import { Provider } from 'react-redux';
import thunk from 'redux-thunk';
import { reducer as form } from 'redux-form';
import { BrowserRouter as Router, Route, Switch } from 'react-router-dom';
import createBrowserHistory from 'history/createBrowserHistory';
import { syncHistoryWithStore, routerReducer as routing } from 'react-router-redux'

// Replace "foo" by the name of the resource type
import foo from './reducers/foo/';
import fooRoutes from './routes/foo';

const store = createStore(
  combineReducers({routing, form, foo}), // Don't forget to register the reducers here
  applyMiddleware(thunk),
);

const history = syncHistoryWithStore(createBrowserHistory(), store);

ReactDom.render(
  <Provider store={store}>
    <Router history={history}>
      <Switch>
        {fooRoutes}
        <Route render={() => <h1>Not Found</h1>}/>
      </Switch>
    </Router>
  </Provider>,
  document.getElementById('root')
);
```

Previous chapter: [Introduction](index.md)

Next chapter: [Vue.js generator](vuejs.md)
