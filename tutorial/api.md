# Creating your First API with API Platform, in 5 Minutes

[API Platform](https://api-platform.com) is one of the most efficient framework out there to create web APIs. It makes it
easy to create simple APIs while giving you the ability to do create complex features with style. To discover the basics,
we will create an API to manage a bookshop.

In only 4 steps, we will create a fully featured API relying on industry leading open standards:

1. Install API Platform
2. Handcraft the data model as *Plain Old PHP Objects*
3. Map those classes with the database
4. Add some validation rules

API Platform will use the data model to expose a read/write web API having a ton of built-in features:

* data validation
* pagination
* filtering
* sorting
* a nice UI and machine-readable documentations ([Swagger/OpenAPI](https://swagger.io), [Hydra](http://hydra-cg.com))
* hypermedia/HATEOAS and content negotiation support ([JSON-LD](http://json-ld.org), [HAL](blog.stateless.co/post/13296666138/json-linking-with-hal))
* authentication ([Basic HTTP](https://en.wikipedia.org/wiki/Basic_access_authentication), cookies as well as [JWT](https://jwt.io/)
  and [OAuth](https://oauth.net/) through extensions)
* [CORS headers support](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS)
* HTTP caching
* and basically everything mandatory for modern APIs.

One more thing, before we start: API Platform is built on top of [the Symfony framework][https://symfony.com]. API Platform
is compatible with most [Symfony bundles](http://symfony.com/blog/the-30-most-useful-symfony-bundles-and-making-them-even-better)
(extensions) and benefits from the numerous extensions provided by this rock-solid foundation (kernel events, DIC...). Adding
features like custom service-oriented API endpoints, JWT or OAuth authentication, HTTP caching, mail sending or asynchronous
jobs to your APIs is very straightforward.

## Installing the framework

API Platform is shipped with a complete [https://docker.com](Docker) setup that makes it easy to get a containerized development
environment up and running. The bundled setup contains an image pre-configured with PHP 7, Apache and everything required
to run API Platform and a MySQL image to host the database.

*Alternatively to using Docker, API Platform can also be installed using [Composer](https://getcomposer.org/):
`composer create-project api-platform/api-platform bookshop-api`.*

Start by [downloading the API Platform Standard Edition](https://api-platform.com/download) and extract the content of the
archive.
The resulting directory contains an empty API Platform project structure. You will add your own code and configuration inside
it.
Then, if you do not already installed Docker on your computer, [it is the right time to do it](https://www.docker.com/products/overview#/install_the_platform).

Open a terminal, go inside the directory containing your project skeleton, and run the following command to start the Apache
and the MySQL servers using Docker Compose:

    $ docker-compose up

The first time you start containers, Docker downloads and builds images for you. It will take some time, but don't worry,
this operation is done only one time. Starting servers will then be lightning fast.

Project's files are automatically shared between your local machine and the container thanks to a pre-configured [Docker
volume](https://docs.docker.com/engine/tutorials/dockervolumes/). It means that you can edit files of your project locally
using your preferred IDE or code editor, they will be transparently taken into account.
Speaking about IDEs, our preferred software to develop API Platform apps is [PHPStorm](https://www.jetbrains.com/phpstorm/)
with its awesome "Symfony" and "PHP annotations" plugins. Give it a try, you will got auto-completion for almost everything.

Now, in another shell, install the PHP dependencies of our project:

    $ docker-compose run web composer install --no-interaction

The `web` container is where your project belongs. Prefixing a command by `docker-compose run web` allow to run the given
command in this container. You may want [to create an alias](http://www.linfo.org/alias.html) to run commands in the container
easily. Here, we installed libraries required by our project using the `composer` command included in the API Platform
image.

The API Platform Standard Edition has a dummy entity for test purpose: `src/AppBundle/Entity/Foo.php`. We will remove it
later, but for now, create the related database table:

    docker-compose run web bin/console doctrine:schema:create

You probably guessed that this test entity uses the industry leading [Doctrine ORM](http://www.doctrine-project.org/projects/orm.html)
library as persistence system.
API Platform is 100% independent of the persistence system and you can use the one(s) that best suit(s) your needs (like
a noSQL database or a remote web service).
API Platform even support using several persistence system together in the same project.

However, Doctrine ORM is definitely the easiest way to persist and query data in an API Platform project thanks to a bridge
included in the Standard Edition. This Doctrine ORM bridge is optimized for performance and development convenience. Doctrine
ORM and its bridge support major RDBMS including MySQL, PostgreSQL, SQLite, SQL Server and MariaDB.

Open `http://localhost` with your favorite web browser:

![Swagger UI integration in API Platform](images/swagger-ui.png)

API Platform exposes a description of the API in the Swagger UI format. It also integrates Swagger UI, a nice interface
rendering the API documentation. Click on an operation to display its details. You can also send requests to the API directly
from the UI. Try to create a new foo object using the `POST` operation, then access it using `GET` and finally  delete it
with `DELETE`.
If you access any API URL using a web browser, API Platform will detect it (using the `Accept` HTTP header send by the header)
and display will the related request to the API in the UI. Open `http://localhost/foos`:

![Request detail in the UI](images/swagger-ui.png)

If you want to access the raw data, you have two alternatives:

* Add the correct `Accept` header (or don't set any `Accept` header at all and API Platform will default to JSON-LD) - preferred
  when writing API clients
* Add the format format you want as the extension of the resource - to debug purpose only

For instance, go to `http://localhost/foos.jsonld` to retrieve the list of Foos object in JSON-LD or http://localhost/foos.json`
to get raw JSON.

You can use your favorite HTTP client to query the API. We strongly recommend to use [Postman](https://www.getpostman.com/).
It works perfectly well with API Platform, has native JSON support, allow you to easily write functional tests for your
API and has very good team collaboration features.

## Creating the model

API Platform is now 100% functional. Let's create our own data model.
Our bookshop API will be simple for now. It will be composed of books and reviews.

Books will have an id, a name, an author, an ISBN number, a description and will be related to a list of reviews.
Reviews

If you want to expose any entity:

* Put it in the `Entity` directory of a bundle
* Mark it with the `@ApiPlatform\Core\Annotation\ApiResource` annotation

It's as easy as it looks.

## Trying the API

Add a person named Olivier Lenancker by issuing a POST request on `http://localhost:8000/people` with the following JSON document as
raw body:

```json
{
  "familyName": "Lenancker",
  "givenName": "Olivier",
  "description": "A famous author from the North.",
  "birthDate": "666-06-06"
}
```

As you can see, we omitted some optional properties such as `description` and `deathDate`.

The data is inserted in database. The server replies with a JSON-LD representation of the freshly created resource.
Thanks to the schema generator, the `@type` property of the JSON-LD document is referencing a Schema.org type:

```json
{
  "@context": "/contexts/Person",
  "@id": "/people/1",
  "@type": "http://schema.org/Person",
  "birthDate": "0666-06-06T00:00:00+00:00",
  "deathDate": null,
  "description": "A famous author from the North.",
  "familyName": "Lenancker",
  "givenName": "Olivier"
}
```

The JSON-LD spec is fully supported by API Platform. Want a proof? Browse `http://localhost:8000/contexts/Person`.

By default, the API allows `GET` (retrieve, on collections and items), `POST` (create), `PUT` (update) and `DELETE` (self-explaining)
HTTP methods. [You can add and remove any other operation you want](../core/operations.md).
Try it!

Now, browse `http://localhost:8000/people`:

```json
{
  "@context": "/contexts/Person",
  "@id": "/people",
  "@type": "hydra:Collection",
  "hydra:member": [
    {
      "@id": "/people/1",
      "@type": "http://schema.org/Person",
      "birthDate": "0666-06-06T00:00:00+00:00",
      "deathDate": null,
      "description": "A famous author from the North.",
      "familyName": "Lenancker",
      "givenName": "Olivier"
    }
  ],
  "hydra:totalItems": 1
}
```

Pagination is also supported (and enabled) out of the box.

It's time to post our first article. Run a POST request on `http://locahost:8000/blog_postings` with the following JSON document
as body:

```json
{
  "name": "API Platform is great",
  "headline": "You'll love that framework!",
  "articleBody": "The body of my article.",
  "articleSection": "technology",
  "author": "/people/1",
  "isFamilyFriendly": "maybe",
  "datePublished": "2015-05-11",
  "kevinReview": "nice"
}
```

Oops... the `isFamilyFriendly` property is a boolean. Our JSON contains an incorrect type value (a `string`).
Fortunately API Platform is smart enough to detect the error: it uses Symfony validation constraints generated previously.
It returns a detailed error message in the Hydra error serialization format:

```json
{
  "@context": "/contexts/ConstraintViolationList",
  "@type": "ConstraintViolationList",
  "hydra:title": "An error occurred",
  "hydra:description": "isFamilyFriendly: This value should be of type boolean.",
  "violations": [
    {
      "propertyPath": "isFamilyFriendly",
      "message": "This value should be of type boolean."
    }
  ]
}
```

Correct the body and send the request again:

```json
{
  "name": "API Platform is great",
  "headline": "You'll love that framework!",
  "articleBody": "The body of my article.",
  "articleSection": "technology",
  "author": "/people/1",
  "isFamilyFriendly": true,
  "datePublished": "2015-05-11",
  "kevinReview": "nice"
}
```

We fixed it! By the way you learned how to work with relations. In a hypermedia API, every resource is identified with
an unique IRI (an URL is an IRI). They are in the `@id` property of every JSON-LD document generated by the API and you
can use it as reference to set relations like we done in the previous snippet for the author property.

API Platform is smart enough to understand [any date format supported by PHP](http://php.net/manual/en/datetime.formats.date.php)
date functions. In production we recommend the format specified by the [RFC 3339](http://tools.ietf.org/html/rfc3339).

We already have a powerful hypermedia REST API (always without writing a single line of PHP), but there is more.

**Our API is auto-discoverable**. Open `http://localhost:8000/apidoc` and take a look at the content. Capabilities of the
API are fully described in a machine-readable format: available resources, properties and operations, description of elements,
readable and writable properties, types returned and expected...

As for errors, the whole API is described using [the Hydra Core Vocabulary](http://www.w3.org/ns/hydra/spec/latest/core/),
an open web standard for describing hypermedia REST APIs in JSON-LD. Any Hydra-compliant client or library is able to interact
with the API without knowing anything about it! The most popular Hydra client is [Hydra Console](http://www.markus-lanthaler.com/hydra/console/).
Open an URL of the API with it you'll get a nice management interface.

![Hydra console](images/console.png)]

You can also give a try to the [hydra-core Javascript library](https://github.com/bergos/hydra-core).

API Platform offers a lot of other features including:

* [filters](../core/filters.md)
* [serialization groups and child resource embedding](../core/serialization-groups-and-relations.md)
* [data providers](../core/data-providers.md): retrieve and modify data trough a web-service or a MongoDB database or anything
  else instead of Doctrine ORM
* [custom operations](../core/operations.md): deactivate some methods, create custom operations, URL and controllers
* a powerful [event system](../core/the-event-system.md)

Read [its dedicated documentation](../core/index.md) to see how to leverage them and how to
hook your own code everywhere into it.

## Specifying and testing the API

[Behat](http://docs.behat.org/) (a [Behavior-driven development](http://en.wikipedia.org/wiki/Behavior-driven_development)
framework) is pre-configured with contexts useful to spec and test REST API and JSON documents.

With Behat, you can write the API specification (as user stories) in natural language then execute scenarios against the
application to validate its behavior.

Create a [Gherkin](http://docs.behat.org/en/latest/user_guide/gherkin.html) feature file containing the scenarios we run manually
in the previous chapter:

```gherkin
# features/blog.feature

Feature: Blog
  In order to post news
  As a client software developer
  I need to be able to retrieve, create, update and delete authors and posts trough the API.

  # "@createSchema" creates a temporary SQLite database for testing the API
  @createSchema
  Scenario: Create a person
    When I send a "POST" request to "/people" with body:
    """
    {
      "familyName": "Lenancker",
      "givenName": "Olivier",
      "description": "A famous author from the North.",
      "birthDate": "666-06-06"
    }
    """
    Then the response status code should be 201
    And the response should be in JSON
    And the header "Content-Type" should be equal to "application/ld+json"
    And the JSON should be equal to:
    """
    {
      "@context": "/contexts/Person",
      "@id": "/people/1",
      "@type": "http://schema.org/Person",
      "birthDate": "0666-06-06T00:00:00+00:00",
      "deathDate": null,
      "description": "A famous author from the North.",
      "familyName": "Lenancker",
      "givenName": "Olivier"
    }
    """

  Scenario: Retrieve the user list
    When I send a "GET" request to "/people"
    Then the response status code should be 200
    And the response should be in JSON
    And the header "Content-Type" should be equal to "application/ld+json"
    And the JSON should be equal to:
    """
    {
      "@context": "/contexts/Person",
      "@id": "/people",
      "@type": "hydra:Collection",
      "hydra:member": [
        {
          "@id": "/people/1",
          "@type": "http://schema.org/Person",
          "birthDate": "0666-06-06T00:00:00+00:00",
          "deathDate": null,
          "description": "A famous author from the North.",
          "familyName": "Lenancker",
          "givenName": "Olivier"
        }
      ],
      "hydra:totalItems": 1
    }
    """

  Scenario: Throw errors when a post is invalid
    When I send a "POST" request to "/blog_postings" with body:
    """
    {
      "name": "API Platform is great",
      "headline": "You'll love that framework!",
      "articleBody": "The body of my article.",
      "articleSection": "technology",
      "author": "/people/1",
      "isFamilyFriendly": "maybe",
      "datePublished": "2015-05-11",
      "kevinReview": "nice"
    }
    """
    Then the response status code should be 400
    And the response should be in JSON
    And the header "Content-Type" should be equal to "application/ld+json"
    And the JSON should be equal to:
    """
    {
      "@context": "/contexts/ConstraintViolationList",
      "@type": "ConstraintViolationList",
      "hydra:title": "An error occurred",
      "hydra:description": "isFamilyFriendly: This value should be of type boolean.",
      "violations": [
        {
          "propertyPath": "isFamilyFriendly",
          "message": "This value should be of type boolean."
        }
      ]
    }
    """

  # "@dropSchema" is mandatory to cleanup the temporary database on the last scenario
  @dropSchema
  Scenario: Post a new blog post
    When I send a "POST" request to "/blog_postings" with body:
    """
    {
      "name": "API Platform is great",
      "headline": "You'll love that framework!",
      "articleBody": "The body of my article.",
      "articleSection": "technology",
      "author": "/people/1",
      "isFamilyFriendly": true,
      "datePublished": "2015-05-11",
      "kevinReview": "nice"
    }
    """
    Then the response status code should be 201
    And the response should be in JSON
    And the header "Content-Type" should be equal to "application/ld+json"
    And the JSON should be equal to:
    """
    {
      "@context": "/contexts/BlogPosting",
      "@id": "/blog_postings/1",
      "@type": "http://schema.org/BlogPosting",
      "articleBody": "The body of my article.",
      "articleSection": "technology",
      "author": "/people/1",
      "datePublished": "2015-05-11T00:00:00+00:00",
      "headline": "You'll love that framework!",
      "isFamilyFriendly": true,
      "name": "API Platform is great",
      "kevinReview": "nice"
    }
    """
```

The API Platform flavor of Behat also comes with a temporary SQLite database dedicated to tests. It works out of the box.

Simply run `vendor/bin/behat`. Everything should be green:

    4 scenarios (4 passed)
    21 steps (21 passed)

Then you get a powerful hypermedia API exposing structured data, specified and tested thanks to Behat. And still without
a line of PHP!

It's incredibly useful for prototyping and Rapid Application Development (RAD). But the framework is designed to run in prod.
It benefits from **strong extension points** and is **has been optimized for very high-traffic websites** (API Platform
powers the new version of a major world-wide media site).

## Other features

API Platform has a lot of other features and can extended with PHP libraries and Symfony bundles. [Stay tuned](https://twitter.com/ApiPlatform),
more documentation and cookbooks are coming!

Here is a non exhaustive list of what you can do with API Platform:

* Add [a user management system](../core/fosuser-bundle.md)
  (FOSUser integration)
* [Secure the API with JWT](https://github.com/lexik/LexikJWTAuthenticationBundle) (LexikJwtAuthenticationBundle) or [OAuth](https://github.com/FriendsOfSymfony/FOSOAuthServerBundle)
  (FosOAuthServer)
* [Add a Varnish reverse proxy and adopt a expiration or invalidation HTTP cache strategy](http://foshttpcachebundle.readthedocs.org)
  (FosHttpCache)
* [Add CSRF protection when the API authentication relies on cookies](https://github.com/dunglas/DunglasAngularCsrfBundle)
  (DunglasAngularCsrfBundle – you should prefer using a stateless authentication mode such as a JWT token stored in the
  browser session storage when possible)
* [Send mails](https://symfony.com/doc/current/cookbook/email/email.html) (Swift Mailer)
* [Deploy](../deployment/index.md)

Keep in mind that you can use your preferred client-side technology: API Platform is tested and approved with React, Angular
1 & 2, Ionic and Swift but can work with any language able to send HTTP requests.
[Checkout our AngularJS client for API Platform tutorial](angularjs.md) to learn how to consume the API with AngularJS.

To go further, the API Platform team maintains a demo application showing more advanced use cases like leveraging serialization
groups, user management or JWT and OAuth authentication. [Checkout the demo code source on GitHub]https://github.com/api-platform/demo)
and [browse it online](https://demo.api-platform.com).

Previous chapter: [Introduction](index.md)
Next chapter: [An AngularJS Client](angularjs.md)
