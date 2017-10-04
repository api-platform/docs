# The API Platform Client Generator

API Platform Client Generator is a generator to scaffold app with Create-Retrieve-Update-Delete features for any API exposing a Hydra documentation for:
 * React/Redux
 * Vue.js

Works especially well with APIs built with the [API Platform](https://api-platform.com) framework.

## Features

* Generate high-quality ES6 components and files built with [React](https://facebook.github.io/react/), [Redux](http://redux.js.org), [React Router](https://reacttraining.com/react-router/) and [Redux Form](http://redux-form.com/) including:
  * A list view
  * A creation form
  * An edition form
  * A deletion button
* Use the Hydra API documentation to generate the code
* Generate the suitable HTML5 input type (`number`, `date`...) according to the type of the API property
* Display of the server-side validation errors under the related input (if using API Platform Core)
* Client-side validation (`required` attributes)
* The generated HTML is compatible with [Bootstrap](https://getbootstrap.com/) and include mandatory classes
* The generated HTML code is accessible to people with disabilities ([ARIA](https://www.w3.org/WAI/intro/aria) support)
* The Redux and the React Router configuration is also generated

Previous chapter: [Admin: Handling Relations to Collections](../admin/handling-relations-to-collections.md)

Next chapter: [React generator](react.md)
