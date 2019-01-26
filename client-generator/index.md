# The API Platform Client Generator

Client Generator is the fastest way to scaffold fully featured webapps and native mobile apps from APIs supporting the [Hydra](http://www.hydra-cg.com/) format.

![Screencast](images/client-generator-demo.gif)

*Generated React and React Native apps, updated in real time* 

It is able to generate apps using the following frontend stacks:

* [React with Redux](react.md)
* [React Native](react-native.md)
* [Vue.js](vuejs.md)

Client Generator works especially well with APIs built with the [API Platform](https://api-platform.com) framework.

## Features

* Generate high-quality ES6:
  * list view (with pagination)
  * detail view
  * creation form
  * editation form
  * delete button
* Supports to-one and to-many relations
* Uses the appropriate input type (`number`, `date`...)
* Client-side validation
* Subscribe to data updates pushed by servers supporting [the Mercure protocol](https://mercure.rocks)
* Display server-side validation errors under the related input (if using API Platform Core)
* Integration with [Bootstrap](https://getbootstrap.com/) and [FontAwesome](https://fontawesome.com/) (Progressive Web Apps)
* Integration with [React Native Elements](https://react-native-training.github.io/react-native-elements/)
* Accessible to people with disabilities ([ARIA](https://www.w3.org/WAI/intro/aria) support in webapps)
