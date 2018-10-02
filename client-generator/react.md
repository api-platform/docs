# React generator

Create a React application using [Facebook's Create React App](https://github.com/facebookincubator/create-react-app):

    $ create-react-app my-app
    $ cd my-app

Install React Router, Redux, React Redux, React Router Redux, Redux Form and Redux Thunk (to handle AJAX requests):

    $ yarn add redux react-redux redux-thunk redux-form react-router-dom react-router-redux prop-types

Install the generator globally:

    $ yarn global add @api-platform/client-generator

Reference the Bootstrap CSS stylesheet in `public/index.html` (optional):

Bootstrap 3 - last release 0.1.15
```html
  <!-- ... -->
    <title>React App</title>

    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous">
  </head>
  <!-- ... -->
```
Bootstrap 4 - from release 0.1.16

```html
  <!-- ... -->
    <title>React App</title>

    <link href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-beta/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-/Y6pD6FV/Vv2HJnA6t+vslU6fwYXjCFtcEpHbNJ0lyAFsXTsjBbfaDjzALeQsN6M" crossorigin="anonymous">
    <link href="https://maxcdn.bootstrapcdn.com/font-awesome/4.7.0/css/font-awesome.min.css" rel="stylesheet" integrity="sha384-wvfXpqpZZVQGK6TAh5PVlGOfQNHSoD2xbE+QkPxCAFlNEevoEH3Sl0sibVcOQVnN" crossorigin="anonymous">
  </head>
  <!-- ... -->
```

In the app directory, generate the files for the resource you want:

```bash
$ generate-api-platform-client https://demo.api-platform.com src/ --resource book
# Replace the URL by the entrypoint of your Hydra-enabled API
# Omit the resource flag to generate files for all resource types exposed by the API
```

The code is ready to be executed! Register the generated reducers and components in the `index.js` file, here is an example:

```javascript
import React from 'react';
import ReactDOM from 'react-dom';
import * as serviceWorker from './serviceWorker';

import { createStore, combineReducers, applyMiddleware } from 'redux';
import { Provider } from 'react-redux';
import thunk from 'redux-thunk';
import { reducer as form } from 'redux-form';
import { BrowserRouter as Router, Route, Switch } from 'react-router-dom';
import createBrowserHistory from 'history/createBrowserHistory';
import { syncHistoryWithStore, routerReducer as routing } from 'react-router-redux'

// Replace "book" with the name of the resource type
import book from './reducers/book/';
import bookRoutes from './routes/book';

const store = createStore(
  combineReducers({routing, form, book}), // Don't forget to register the reducers here
  applyMiddleware(thunk),
);

const history = syncHistoryWithStore(createBrowserHistory(), store);

ReactDOM.render(
  <Provider store={store}>
    <Router history={history}>
      <Switch>
        {bookRoutes}
        <Route render={() => <h1>Not Found</h1>}/>
      </Switch>
    </Router>
  </Provider>,
  document.getElementById('root')
);

// If you want your app to work offline and load faster, you can change
// unregister() to register() below. Note this comes with some pitfalls.
// Learn more about service workers: http://bit.ly/CRA-PWA
serviceWorker.unregister();
```
