# Admin On REST generator

## Summary
This generator is alternative to api-platform/admin [api-platform/admin](https://github.com/api-platform/admin)

**api-platform/admin** allows to generate all resources on the fly and is immune to API changes.

**Generator** allows more configuration to the resource components by generated config file (config file per resource). In case of API changes resource components have to be regenerated to expose changes

Example configuration of the resource component at the bottom of this document.

## Usage
Create a React application using [Facebook's Create React App](https://github.com/facebookincubator/create-react-app):

    $ create-react-app my-app
    $ cd my-app

React and React DOM will be directly provided as dependencies of Admin On REST. As having different versions of React
causes issues, remove `react` and `react-dom` from the `dependencies` section of the generated `package.json` file:

```patch
-    "react": "^16.0.0",
-    "react-dom": "^16.0.0",
```
Install admin-on-rest

    $ yarn add admin-on-rest

Install the generator globally:

    $ yarn global add @api-platform/client-generator

In the app directory, generate the files for the resource you want:

    $ generate-api-platform-client https://demo.api-platform.com src/ -g admin-on-rest --resource foo
    # Replace the URL by the entrypoint of your Hydra-enabled API
    # Omit the resource flag to generate files for all resource types exposed by the API

Create *apiPlatformRestClient.js*

```javascript
import {
  GET_LIST,
  GET_ONE,
  GET_MANY,
  GET_MANY_REFERENCE,
  CREATE,
  UPDATE,
  DELETE,
  fetchUtils
} from 'admin-on-rest';
import {stringify} from 'query-string';

const {fetchJson} = fetchUtils;

export default (API_URL, authTokenName = null, authTokenValue = null, httpClient = fetchJson) => {
  /**
   * @param {String} type One of the constants appearing at the top if this file, e.g. 'UPDATE'
   * @param {String} resource Name of the resource to fetch, e.g. 'posts'
   * @param {Object} params The REST request params, depending on the type
   * @returns {Object} { url, options } The HTTP request parameters
   */
  const convertRESTRequestToHTTP = (type, resource, params) => {
    let url = '';
    const options = {};
    options.headers = new Headers({'Accept': 'application/ld+json'});
    if( authTokenName && authTokenValue) {
      let authHeader = {};
      authHeader[authTokenName] = authTokenValue;
      options.headers = new Headers(authHeader);
    }
    switch (type) {
      case GET_LIST: {
        const {page, perPage} = params.pagination;
        url = `${API_URL}/${resource}?page=${page}&itemsPerPage=${perPage}`;
        break;
      }
      case GET_ONE:
        url = `${API_URL}/${resource}/${params.id}`;
        break;
      case GET_MANY: {
        const query = {
          filter: JSON.stringify({id: params.ids}),
        };
        url = `${API_URL}/${resource}?${stringify(query)}`;
        break;
      }
      case GET_MANY_REFERENCE: {
        const {page, perPage} = params.pagination;
        const {field, order} = params.sort;
        const query = {
          sort: JSON.stringify([field, order]),
          range: JSON.stringify([(page - 1) * perPage, (page * perPage) - 1]),
          filter: JSON.stringify({...params.filter, [params.target]: params.id}),
        };
        url = `${API_URL}/${resource}?${stringify(query)}`;
        break;
      }
      case UPDATE:
        url = `${API_URL}/${resource}/${params.id}`;
        options.method = 'PUT';
        options.body = JSON.stringify(params.data);
        break;
      case CREATE:
        url = `${API_URL}/${resource}`;
        options.method = 'POST';
        options.body = JSON.stringify(params.data);
        break;
      case DELETE:
        url = `${API_URL}/${resource}/${params.id}`;
        options.method = 'DELETE';
        break;
      default:
        throw new Error(`Unsupported fetch action type ${type}`);
    }
    return { url, options };
  };

  /**
   * @param {Object} response HTTP response from fetch()
   * @param {String} type One of the constants appearing at the top if this file, e.g. 'UPDATE'
   * @param {String} resource Name of the resource to fetch, e.g. 'posts'
   * @param {Object} params The REST request params, depending on the type
   * @returns {Object} REST response
   */
  const convertHTTPResponseToREST = (response, type, resource, params) => {
    const { json } = response;
    switch (type) {
      case GET_LIST:
      case GET_MANY_REFERENCE:
        return {
          data: json["hydra:member"].map(x => x),
          total: json["hydra:totalItems"]
        };
      case CREATE:
        return { data: { ...params.data, id: json.id } };
      case DELETE:
        return { data: { ...params.data } };
      default:
        return { data: json };
    }
  };

  /**
   * @param {string} type Request type, e.g GET_LIST
   * @param {string} resource Resource name, e.g. "posts"
   * @param {Object} payload Request parameters. Depends on the request type
   * @returns {Promise} the Promise for a REST response
   */
  return (type, resource, params) => {
    // json-server doesn't handle WHERE IN requests, so we fallback to calling GET_ONE n times instead
    if (type === GET_MANY) {
      return Promise.all(params.ids.map(id => httpClient(`${API_URL}/${resource}/${id}`)))
      .then(responses => ({ data: responses.map(response => response.json) }));
    }
    const { url, options } = convertRESTRequestToHTTP(type, resource, params);
    return httpClient(url, options)
    .then(response => convertHTTPResponseToREST(response, type, resource, params));
  };
};
```

Replace App.js content with

```javascript
import React, { Component } from 'react';
import { Admin } from "admin-on-rest";
import restClient from './apiPlatformRestClient';
import { API_HOST, API_PATH } from "./config/_entrypoint";
import  * as resources from "./resource-import";

const API_URL = (API_HOST + API_PATH).replace(/\/$/, "");

class App extends Component {
  render() {
    return (
      <Admin restClient={restClient(API_URL)}>
        {Object.keys(resources).map( (key) => resources[key] )}
      </Admin>
    );
  }
}

export default App;


````
The code is ready to be executed!

Resource configuration example config/book.js:

Each resource component has fields and buttons config options.
False setting hides field or button.
```javascript
export const configList = {
  '@id': true,           
  id: true,              
  isbn: true,            
  description: true,      
  author: true,          
  title: true,           
  publicationDate: true, 
  buttons: {
    show: true,
    edit: true,
    create: true,
    refresh: true,
    delete: true,
  }
}

export const configEdit = {
  '@id': true,
  id: true,
  isbn: true,
  description: true,
  author: true,
  title: true,
  publicationDate: true,
  buttons: {
    show: true,
    list: true,
    delete: true,
    refresh: true,
  }
}

export const configCreate = {
  '@id': true,
  id: true,
  isbn: true,
  description: true,
  author: true,
  title: true,
  publicationDate: true,
  buttons: {
    list: true,
  }
}

export const configShow = {
  '@id': true,
  id: true,
  isbn: true,
  description: true,
  author: true,
  title: true,
  publicationDate: true,
  buttons: {
    edit: true,
    list: true,
    delete: true,
    refresh: true,
  }
}

```
