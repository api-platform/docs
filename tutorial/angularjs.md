# An AngularJS Client

Prerequisites:

* Having finished [the API tutorial](api.md)
* [Node.js](https://nodejs.org/) and [NPM](https://www.npmjs.com/) installed.

API Platform is agnostic of the client-side technology. You can use whatever platform, language or framework you want. As
an illustration of doors opened by an API-first architecture and the ease of development of SPA, we will create a tiny AngularJS
client.

Keep in mind that **this is not an AngularJS tutorial** and it doesn't pretend to follow AngularJS best practices. It's
just an example of how simple application development become when all the business logic is encapsulated in an API. The
unique responsibility of our AngularJS blog client is to display informations retrieved by the API (presentation layer).

The AngularJS application is fully independent of the API. It's a HTML/JS/CSS app querying an API trough AJAX. It leaves
in its own Git repository and is hosted on its own web server. As it only contains assets, it can even be hosted directly
on a CDN such as Amazon CloudFront or Akamai.

To scaffold our AngularJS app we will use the official [Angular generator](https://github.com/yeoman/generator-angular)
of [Yeoman](http://yeoman.io/). Install the generator then generate a client skeleton:

    mkdir blog-client
    cd blog-client
    yo angular blog

Yeoman will ask some questions. We want to keep the app minimal:

* choose Grunt or Gulp (experimental), both will work
* don't install Sass support
* install Twitter Bootstrap (it's optional but the app will look better)
* uncheck all suggested angular modules

However, we will install [Restangular](https://github.com/mgonto/restangular), an awesome REST client for Angular that fit
well with API Platform:

    bower install --save lodash restangular

The Angular generator comes with the [Grunt](http://gruntjs.com/) build tool. It compiles assets (minification, compression)
and can serve the web app with an integrated development web server. Start it:

    grunt serve

DunglasApiBundle provides [a Restangular integration guide](https://github.com/dunglas/DunglasApiBundle/blob/master/Resources/doc/angular-integration.md)
in its documentation. Once configured, Restangular will work with the JSON-LD/Hydra API like with any other standard REST
API.

Then edit `app/app.js` file to register Restangular and configure it properly as explained [in the dedicated documentation
chapter](../core/angular-integration.md):

```javascript
// app/scripts/app.js
 
'use strict';
 
/**
 * @ngdoc overview
 * @name blogApp
 * @description
 * # blogApp
 *
 * Main module of the application.
 */
angular
    .module('blogApp', ['restangular'])
    .config(['RestangularProvider', function (RestangularProvider) {
        // The URL of the API endpoint
        RestangularProvider.setBaseUrl('http://localhost:8000');

        // JSON-LD @id support
        RestangularProvider.setRestangularFields({
            id: '@id',
            selfLink: '@id'
        });
        RestangularProvider.setSelfLinkAbsoluteUrl(false);

        // Hydra collections support
        RestangularProvider.addResponseInterceptor(function (data, operation) {
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
                angular.forEach(data, function (value, key) {
                    if ('hydra:member' !== key) {
                        collectionResponse.metadata[key] = value;
                    }
                });

                // Populate href property for all elements of the collection
                angular.forEach(collectionResponse, function (value) {
                    populateHref(value);
                });

                return collectionResponse;
            }

            return data;
        });
    }])
;
```

Be sure to change the base URL with the one of your API in production (_protip_: use [grunt-ng-constant](https://github.com/werk85/grunt-ng-constant)).

And here is the controller retrieving the posts list and allowing to create a new one:

```javascript
// app/scripts/controllers/main.js

'use strict';

/**
 * @ngdoc function
 * @name blogApp.controller:MainCtrl
 * @description
 * # MainCtrl
 * Controller of the blogApp
 */
angular.module('blogApp')
    .controller('MainCtrl', function ($scope, Restangular) {
        var blogPostingApi = Restangular.all('blog_postings');
        var peopleApi = Restangular.all('people');

        function loadPosts() {
            blogPostingApi.getList().then(function (posts) {
                $scope.posts = posts;
            });
        }

        loadPosts();
        peopleApi.getList().then(function (people) {
            $scope.people = people;
        });

        $scope.newPost = {};
        $scope.success = false;
        $scope.errorTitle = false;
        $scope.errorDescription = false;

        $scope.createPost = function (form) {
            blogPostingApi.post($scope.newPost).then(function () {
                loadPosts();

                $scope.success = true;
                $scope.errorTitle = false;
                $scope.errorDescription = false;

                $scope.newPost = {};
                form.$setPristine();
            }, function (response) {
                $scope.success = false;
                $scope.errorTitle = response.data['hydra:title'];
                $scope.errorDescription = response.data['hydra:description'];
            });
        };
    })
;
```

As you can see, querying the API with Restangular is easy and very intuitive. The library automatically issues HTTP requests
to the server and hydrates "magic" JavaScript objects corresponding to JSON responses that can be manipulated to modify
remote resources (trough `PUT`, `POST` and `DELETE` requests).

We also leverage the server side error handling to display beautiful messages when submitted data are invalid or when something
goes wrong.

And the corresponding view looping over posts and displaying the form and errors:

```html
<!-- app/views/main.html -->

<!-- ... -->

<article ng-repeat="post in posts" id="{{ post['@id'] }}" class="row marketing">
    <h1>{{ post.name }}</h1>
    <h2>{{ post.headline }}</h2>

    <header>
        Date: {{ post.datePublished | date:'medium' }}
        <span ng-hide="post.isFamilyFriendly"> - <b>NSFW</b></span>
    </header>

    <p>{{ post.articleBody }}</p>

    <footer>
        Section: {{ post.articleSection }}
    </footer>
</article>

<form name="createPostForm" ng-submit="createPost(createPostForm)" class="row marketing">
    <h1>Post a new article</h1>

    <div ng-show="success" class="alert alert-success" role="alert">Post published.</div>
    <div ng-show="errorTitle" class="alert alert-danger" role="alert">
        <b>{{ errorTitle }}</b><br>

        {{ errorDescription }}
    </div>

    <div class="form-group">
        <input ng-model="newPost.name" placeholder="Name" class="form-control">
    </div>

    <div class="form-group">
        <input ng-model="newPost.headline" placeholder="Headline" class="form-control">
    </div>

    <div class="form-group">
        <textarea ng-model="newPost.articleBody" placeholder="Body" class="form-control"></textarea>
    </div>

    <div class="form-group">
        <label for="author">Author</label>
        <select ng-model="newPost.author" ng-options="person['@id'] as person.name for person in people" id="author">
        </select>
    </div>

    <div class="form-group">
        <input ng-model="newPost.datePublished" placeholder="Date published" class="form-control">
    </div>

    <div class="form-group">
        <input ng-model="newPost.articleSection" placeholder="Section" class="form-control">
    </div>

    <div class="checkbox">
        <label>
            <input type="checkbox" ng-model="newPost.isFamilyFriendly"> is family friendly?
        </label>
    </div>

    <button type="submit" class="btn btn-primary">Submit</button>
</form>
```

It's the end of this tutorial. Our blog client is ready and working! Remember of the old way of doing synchronous web apps?
What do you think of this new approach? Easier and very much powerful isn't it?

Of course there are some tasks to have a finished client including routing, pagination support and injecting raw JSON-LD
in the generated HTML for search engines (use [the response extractor hook](https://github.com/mgonto/restangular/issues/100)
provided by Restangular). As it's only Angular tasks, it's out of scope of this introduction to API Platform. But it's a
good exercise to add such features to the client. Feel free to share your snippets!

Previous chapter: [Creating your First API with API Platform, in 5 Minutes](api.md)<br>
Next chapter: [API Platform Core: Introduction](../core/index.md)
