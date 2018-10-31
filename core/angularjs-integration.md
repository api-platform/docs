# AngularJS Integration

Warning: For new project, you should consider using [the API Platform's Progressive Web App generator](../client-generator/index.md)
(that supports React and Vue.js) instead of this Angular v1 integration.

## Restangular

API Platform works fine with [AngularJS v1](http://angularjs.org). The popular [Restangular](https://github.com/mgonto/restangular)
REST client library for Angular can easily be configured to handle the API format.

Here is a working Restangular config:

```javascript
'use strict';

var app = angular
    .module('myAngularjsApp')
    .config(['RestangularProvider', function(RestangularProvider) {
        // The URL of the API endpoint
        RestangularProvider.setBaseUrl('http://localhost:8000');

        // JSON-LD @id support
        RestangularProvider.setRestangularFields({
            id: '@id',
            selfLink: '@id'
        });
        RestangularProvider.setSelfLinkAbsoluteUrl(false);

        // Hydra collections support
        RestangularProvider.addResponseInterceptor(function(data, operation) {
            // Remove trailing slash to make Restangular working
            function populateHref(data) {
                if (data['@id']) {
                    data.href = data['@id'].substring(1);
                }
            }

            // Populate href property for the collection
            populateHref(data);

            if ('getList' === operation) {
                var collectionResponse = data['hydra:member'];
                collectionResponse.metadata = {};

                // Put metadata in a property of the collection
                angular.forEach(data, function(value, key) {
                    if ('hydra:member' !== key) {
                        collectionResponse.metadata[key] = value;
                    }
                });

                // Populate href property for all elements of the collection
                angular.forEach(collectionResponse, function(value) {
                    populateHref(value);
                });

                return collectionResponse;
            }

            return data;
        });
    }]);
```

## ng-admin

If you want to use [ng-admin](https://github.com/marmelab/ng-admin), set the [Restangular](#restangular) config,
then create your entities like in the following example :

```javascript
'use strict';

var nga = NgAdminConfigurationProvider;

var admin = nga
    .application('My First Admin')
    .baseApiUrl('http://localhost:8000');

var article = nga.entity('articles');
article.identifier(nga.field('@id'));
article.url(function(entityName, viewType, identifierValue) {
    var url = '/' + entityName;

    if (viewType === 'ListView' || viewType === 'CreateView') {
        return url;
    }

    return identifierValue ? decodeURIComponent(identifierValue) : url;
});

article.listView().fields([
    nga.field('title'),
    nga.field('content')
]);

admin.addEntity(article);
nga.configure(admin);
```

You can look at what we have done as another example [api-platform/admin](https://github.com/api-platform/admin).
