# Creating your First API with API Platform, in 5 Minutes

[API Platform](https://api-platform.com) is one of the most efficient framework out there to create web APIs. It makes it
easy to start creating APIs with the support of industry-leading open standards, while giving you the flexibility to build
complex features.
To discover the basics, we will create an API to manage a bookshop.

In a few minutes and just 2 steps, we will create a fully featured API:

1. Install API Platform
2. Handcraft the API data model as *Plain Old PHP Objects*

API Platform uses these model classes to expose a web API having a ton of built-in features:

* creating, retrieving, updating and deleting (CRUD) resources
* data validation
* pagination
* filtering
* sorting
* a nice UI and machine-readable documentation ([Swagger/OpenAPI](https://swagger.io), [Hydra](http://hydra-cg.com))
* hypermedia/[HATEOAS](https://en.wikipedia.org/wiki/HATEOAS) and content negotiation support ([JSON-LD](http://json-ld.org),
[HAL](http://blog.stateless.co/post/13296666138/json-linking-with-hal))
* authentication ([Basic HTTP](https://en.wikipedia.org/wiki/Basic_access_authentication), cookies as well as [JWT](https://jwt.io/)
  and [OAuth](https://oauth.net/) through extensions)
* [CORS headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS)
* security (tested against [OWASP recommendations](https://www.owasp.org/index.php/REST_Security_Cheat_Sheet))
* HTTP caching
* and basically everything needed to build modern APIs.

One more thing, before we start: API Platform is built on top of [the Symfony framework](https://symfony.com). API Platform
is compatible with most [Symfony bundles](https://symfony.com/blog/the-30-most-useful-symfony-bundles-and-making-them-even-better)
(plugins) and benefits from the numerous extensions points provided by this rock-solid foundation (events, DIC...).
Adding features like custom, service-oriented, API endpoints, JWT or OAuth authentication, HTTP caching, mail sending or
asynchronous jobs to your APIs is very straightforward.

## Installing the framework

### In Docker container

API Platform is shipped with a [Docker](https://docker.com) setup that makes it easy to get a containerized development
environment up and running. This setup contains an image pre-configured with PHP 7, Apache and everything needed to run API
Platform and a MySQL image to host the database.

Start by [downloading the API Platform Standard Edition archive](https://api.github.com/repos/api-platform/api-platform/zipball) and extract its content.
The resulting directory contains an empty API Platform project structure. You will add your own code and configuration inside
it.
Then, if you do not already have Docker on your computer, [it's the right time to install it](https://www.docker.com/products/overview#/install_the_platform).

Open a terminal, and navigate to the directory containing your project skeleton. Then, run the following command to start
Apache and MySQL using [Docker Compose](https://docs.docker.com/compose/):

    $ docker-compose up -d # Running in detached mode

If you encounter problems running Docker on Windows (especially with Docker Toolbox), see [our Troubleshooting guide](../troubleshooting.md#using-docker).

The first time you start the containers, Docker downloads and builds images for you. It will take some time, but don't worry,
this is done only once. Starting servers will then be lightning fast.

In order to see container's logs you will have to do:

    $ docker-compose logs -f # follow the logs

Project's files are automatically shared between your local host machine and the container thanks to a pre-configured [Docker
volume](https://docs.docker.com/engine/tutorials/dockervolumes/). It means that you can edit files of your project locally
using your IDE or code editor, they will be transparently taken into account in the container.
Speaking about IDEs, our favorite software to develop API Platform apps is [PHPStorm](https://www.jetbrains.com/phpstorm/)
with its awesome [Symfony](https://confluence.jetbrains.com/display/PhpStorm/Getting+Started+-+Symfony+Development+using+PhpStorm)
and [PHP annotations](https://plugins.jetbrains.com/plugin/7320) plugins. Give them a try, you'll got auto-completion for
almost everything.

The API Platform Standard Edition comes with a dummy entity for test purpose: `src/AppBundle/Entity/Foo.php`. We will remove
it later, but for now, create the related database table:

    $ docker-compose exec app bin/console doctrine:schema:create

The `app` container is where your project stands. Prefixing a command by `docker-compose exec app` allows to execute the
given command in the container. You may want [to create an alias](http://www.linfo.org/alias.html) to easily run commands
inside the container.

If you're used to the PHP ecosystem, you probably guessed that this test entity uses the industry-leading [Doctrine ORM](http://www.doctrine-project.org/projects/orm.html)
library as persistence system.
API Platform is 100% independent of the persistence system and you can use the one(s) that best suit(s) your needs (like
a NoSQL database or a remote web service).
API Platform even supports using several persistence systems together in the same project.

However, Doctrine ORM is definitely the easiest way to persist and query data in an API Platform project thanks to a bridge
included in the Standard Edition. This Doctrine ORM bridge is optimized for performance and development convenience. Doctrine
ORM and its bridge supports major RDBMS including MySQL, PostgreSQL, SQLite, SQL Server and MariaDB.

### Via Composer

Instead of using Docker, API Platform can also be installed on the local machine using [Composer](https://getcomposer.org/):

    $ composer create-project api-platform/api-platform bookshop-api
    
Then, enter the project folder, create the database and its schema:  

    $ cd bookshop-api
    $ bin/console doctrine:database:create
    $ bin/console doctrine:schema:create
    
And start the server:    

    $ bin/console server:run

## It's ready!

Open `http://localhost` with your favorite web browser:

![Swagger UI integration in API Platform](images/swagger-ui-1.png)

API Platform exposes a description of the API in the Swagger format. It also integrates Swagger UI, a nice interface rendering
the API documentation. Click on an operation to display its details. You can also send requests to the API directly from the UI.
Try to create a new *Foo* resource using the `POST` operation, then access it using the `GET` operation and, finally, delete
it by executing the `DELETE` operation.
If you access any API URL using a web browser, API Platform detects it (using the `Accept` HTTP header) and displays the
corresponding API request in the UI. Open `http://localhost/foos`:

![Request detail in the UI](images/swagger-ui-2.png)

If you want to access the raw data, you have two alternatives:

* Add the correct `Accept` header (or don't set any `Accept` header at all and API Platform will default to JSON-LD) - preferred
  when writing API clients
* Add the format you want as the extension of the resource - for debug purpose only

For instance, go to `http://localhost/foos.jsonld` to retrieve the list of `Foo` resources in JSON-LD or `http://localhost/foos.json`
to retrieve data in raw JSON.

Of course, you can also use your favorite HTTP client to query the API. We strongly recommend to use [Postman](https://www.getpostman.com/).
It works perfectly well with API Platform, also has native Swagger support, allows to easily write functional tests and
has very good team collaboration features.

## Creating the model

API Platform is now 100% functional. Let's create our own data model.
Our bookshop API will start simple. It will be composed of a `Book` resource type and a `Review` one.

Books have an id, an ISBN number, a title, a description, an author, a publication date and are related to a list of reviews.
Reviews have an id, a rating (between 0 and 5), a body, an author, a publication date and are related to one book.

Let's describe this data model as a set of Plain Old PHP Objects (POPO) and map it to database's tables using annotations
provided by the Doctrine ORM:


```php
<?php

// src/AppBundle/Entity/Book.php

namespace AppBundle\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * A book.
 *
 * @ORM\Entity
 */
class Book
{
    /**
     * @var int The id of this book.
     *
     * @ORM\Id
     * @ORM\GeneratedValue
     * @ORM\Column(type="integer")
     */
    private $id;

    /**
     * @var string|null The ISBN number of this book (or null if doesn't have one).
     *
     * @ORM\Column(nullable=true)
     */
    private $isbn;

    /**
     * @var string The title of this book.
     *
     * @ORM\Column
     */
    private $title;

    /**
     * @var string The description of this book.
     *
     * @ORM\Column(type="text")
     */
    private $description;

    /**
     * @var string The author of this book.
     *
     * @ORM\Column
     */
    private $author;

    /**
     * @var \DateTimeInterface The publication date of this book.
     *
     * @ORM\Column(type="datetime")
     */
    private $publicationDate;

    /**
     * @var Review[] Available reviews for this book.
     *
     * @ORM\OneToMany(targetEntity="Review", mappedBy="book")
     */
    private $reviews;
}
```

```php
<?php

// src/AppBundle/Entity/Review.php

namespace AppBundle\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * A review of a book.
 *
 * @ORM\Entity
 */
class Review
{
    /**
     * @var int The id of this review.
     *
     * @ORM\Id
     * @ORM\GeneratedValue
     * @ORM\Column(type="integer")
     */
    private $id;

    /**
     * @var int The rating of this review (between 0 and 5).
     *
     * @ORM\Column(type="smallint")
     */
    private $rating;

    /**
     * @var string the body of the review.
     *
     * @ORM\Column(type="text")
     */
    private $body;

    /**
     * @var string The author of the review.
     *
     * @ORM\Column
     */
    private $author;

    /**
     * @var \DateTimeInterface The date of publication of this review.
     *
     * @ORM\Column(type="datetime")
     */
    private $publicationDate;

    /**
     * @var Book The book this review is about.
     *
     * @ORM\ManyToOne(targetEntity="Book", inversedBy="reviews")
     */
    private $book;
}
```

As you can see there are two typical PHP objects with the corresponding PHPDoc (note that entities's and properties's descriptions
included in their PHPDoc will appear in the API documentation).

Doctrine's annotations map these entities to tables in the MySQL database. Annotations are convenient as
they allow grouping the code and the configuration but, if you want to decouple classes from their metadata, you can switch
to XML or YAML mappings. They are supported as well.

Learn more about how to map entities with the Doctrine ORM in [the project's official documentation](http://docs.doctrine-project.org/projects/doctrine-orm/en/latest/reference/association-mapping.html)
or in Kévin's book "[Persistence in PHP with the Doctrine ORM](https://www.amazon.fr/gp/product/B00HEGSKYQ/ref=as_li_tl?ie=UTF8&camp=1642&creative=6746&creativeASIN=B00HEGSKYQ&linkCode=as2&tag=kevidung-21)".

As we used private properties (but API Platform as well as Doctrine can also work with public ones), we need to create the
corresponding accessor methods. Run the following command or use the code generation feature of your IDE to generate them:

    $ docker-compose exec app bin/console doctrine:generate:entities AppBundle

Then, delete the file `src/AppBundle/Entity/Foo.php`, this demo entity isn't useful anymore.
Finally, tell Doctrine to sync the database's tables structure with our new data model:

    $ docker-compose exec app bin/console doctrine:schema:update --force

We now have a working data model that you can persist and query. To create an API endpoint with CRUD capabilities corresponding
to an entity class, we just have to mark it with an annotation called `@ApiResource`:

```php
<?php

// src/AppBundle/Entity/Book.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * A book.
 *
 * @ApiResource
 * @ORM\Entity
 */
class Book
{
    // ...
}
```

```php
<?php

// src/AppBundle/Entity/Entity.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * A review of a book.
 *
 * @ApiResource
 * @ORM\Entity
 */
class Review
{
    // ...
}
```

**Our API is (almost) ready!**
Browse `http://localhost/app_dev.php` to load the development environment (including the awesome [Symfony profiler](https://symfony.com/blog/new-in-symfony-2-8-redesigned-profiler)).

Operations available for our 2 resources types appear in the UI.

Click on the `POST` operation of the `Book` resource type and send the following JSON document as request body:

```json
{
  "isbn": "9781782164104",
  "title": "Persistence in PHP with the Doctrine ORM",
  "description": "This book is designed for PHP developers and architects who want to modernize their skills through better understanding of Persistence and ORM.",
  "author": "Kévin Dunglas",
  "publicationDate": "2013-12-01"
}
```

You just saved a new book resource through the bookshop API! API Platform automatically transforms the JSON document to
an instance of the corresponding PHP entity class and uses Doctrine ORM to persist it in the database.

By default, the API supports `GET` (retrieve, on collections and items), `POST` (create), `PUT` (update) and `DELETE` (self-explaining)
HTTP methods. You are not limited to the built-in operations. You can [add new custom operations](../core/operations.md#creating-custom-operations-and-controllers)
(`PATCH` operations, sub-resources...) or [disable the ones you don't want](../core/operations.md#enabling-and-disabling-operations).

Try the `GET` operation on the collection. The book we added appears. When the collection will contain more than 30 items,
the pagination will automatically show up, [and this is entirely configurable](../core/pagination.md). You may be interested
in [adding some filters and adding sorts to the collection](../core/filters.md) as well.

You may have noticed that some keys start with the `@` symbol in the generated JSON response (`@id`, `@type`, `@context`...)?
API Platform comes with a full support of the [JSON-LD](http://json-ld.org/) format (and its [Hydra](http://www.hydra-cg.com/)
extension). It allows to build smart clients, with auto-discoverability capabilities (take a look at [Hydra console](http://www.markus-lanthaler.com/hydra/console/))
and is very useful for open data, SEO and interoperability when [used with open vocabularies such as Schema.org](http://blog.schema.org/2013/06/schemaorg-and-json-ld.html).
JSON-LD enables a lot of awesome advanced features (like [giving access to Google to your structured data](https://developers.google.com/search/docs/guides/intro-structured-data)
or consuming APIs with [Apache Jena](https://jena.apache.org/documentation/io/#formats)).
We think that it's the best default format for a new API. However, API Platform natively [supports many other formats](../core/content-negotiation.md)
including [HAL](http://stateless.co/hal_specification.html), raw [JSON](http://www.json.org/), [XML](https://www.w3.org/XML/)
(experimental) and even [YAML](http://yaml.org/) and [CSV](https://en.wikipedia.org/wiki/Comma-separated_values) (if you
use Symfony 3.2+).
It's up to you to choose which format to enable and to use by default. You can also easily [add support for other formats](../core/content-negotiation.md)
if you need to.

Now, add a review for this book using the `POST` operation for the `Review` resource:

```json
{
    "book": "/books/1",
    "rating": 5,
    "body": "Interesting book!",
    "author": "Kévin",
    "publicationDate": "September 21, 2016"
}
```

There are two interesting things to mention about this request:

First, we learned how to work with relations. In a hypermedia API, every resource is identified by an (unique) [IRI](https://en.wikipedia.org/wiki/Internationalized_Resource_Identifier).
A URL is a valid IRI, and it's what API Platform uses. The `@id` property of every JSON-LD document contains the IRI identifying
it. You can use this IRI to reference this document from other documents. In the previous request, we used the IRI of the
book we created earlier to link it with the `Review` we were creating. API Platform is smart enough to deal with IRIs.
By the way, you may want to [embed documents](../core/serialization-groups-and-relations.md) instead of referencing them
(e.g. to reduce the number of HTTP requests).

The other interesting thing is how API Platform handles dates (the `publicationDate` property). API Platform understands
[any date format supported by PHP](http://php.net/manual/en/datetime.formats.date.php). In production we strongly recommend
to use the format specified by the [RFC 3339](http://tools.ietf.org/html/rfc3339), but, as you can see, most common formats
including `September 21, 2016` can be used.

To summarize, if you want to expose any entity you just have to:

1. Put it in the `Entity` directory of a bundle
2. If you use Doctrine, map it with the database
3. Mark it with the `@ApiPlatform\Core\Annotation\ApiResource` annotation

How can it be more easy?!

## Validating Data

Now try to add another book by issuing a `POST` request to `/books` with the following body:

```json
{
  "isbn": "2815840053",
  "description": "Hello",
  "author": "Me",
  "publicationDate": "today"
}
```

Oops, we missed to add the title. But submit the request anyway. You should get a 500 error with the following message:

    An exception occurred while executing 'INSERT INTO book [...] VALUES [...]' with params [...]:
    SQLSTATE[23000]: Integrity constraint violation: 1048 Column 'title' cannot be null

Did you notice that the error was automatically serialized in JSON-LD and respect the Hydra Core vocabulary for errors?
It allows the client to easily extract useful information from the error. Anyway, it's bad to get a SQL error when submitting
a request. It means that we doesn't use a valid input, and [it's a very bad and dangerous practice](https://www.owasp.org/index.php/Input_Validation_Cheat_Sheet).

API Platform comes with a bridge with [the Symfony Validator Component](http://symfony.com/doc/current/validation.html).
Adding some of [its numerous validation constraints](http://symfony.com/doc/current/validation.html#supported-constraints)
(or [creating custom ones](http://symfony.com/doc/current/validation/custom_constraint.html)) to our entities is enough
to get validate user submitted data. Let's add some validation rules to our data model:

```php
<?php

// src/AppBundle/Entity/Book.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * A book.
 *
 * @ApiResource
 * @ORM\Entity
 */
class Book
{
    /**
     * @var int The id of this book.
     *
     * @ORM\Id
     * @ORM\GeneratedValue
     * @ORM\Column(type="integer")
     */
    private $id;

    /**
     * @var string|null The ISBN number of this book (or null if doesn't have one).
     *
     * @ORM\Column(nullable=true)
     * @Assert\Isbn
     */
    private $isbn;

    /**
     * @var string The title of this book.
     *
     * @ORM\Column
     * @Assert\NotBlank
     */
    private $title;

    /**
     * @var string The description of this book.
     *
     * @ORM\Column(type="text")
     * @Assert\NotBlank
     */
    private $description;

    /**
     * @var string The author of this book.
     *
     * @ORM\Column
     * @Assert\NotBlank
     */
    private $author;

    /**
     * @var \DateTimeInterface The publication date of this book.
     *
     * @ORM\Column(type="datetime")
     * @Assert\NotNull
     */
    private $publicationDate;

    /**
     * @var Review[] Available reviews for this book.
     *
     * @ORM\OneToMany(targetEntity="Review", mappedBy="book")
     */
    private $reviews;

    // ...
}
```

```php
<?php

// src/Entity/Review.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * A review of a book.
 *
 * @ApiResource
 * @ORM\Entity
 */
class Review
{
    /**
     * @var int The id of this review.
     *
     * @ORM\Id
     * @ORM\GeneratedValue
     * @ORM\Column(type="integer")
     */
    private $id;

    /**
     * @var int The rating of this review (between 0 and 5).
     *
     * @ORM\Column(type="smallint")
     * @Assert\Range(min=0, max=5)
     */
    private $rating;

    /**
     * @var string the body of the review.
     *
     * @ORM\Column(type="text")
     * @Assert\NotBlank
     */
    private $body;

    /**
     * @var string The author of the review.
     *
     * @ORM\Column
     * @Assert\NotBlank
     */
    private $author;

    /**
     * @var \DateTimeInterface The date of publication of this review.
     *
     * @ORM\Column(type="datetime")
     * @Assert\NotBlank
     */
    private $publicationDate;

    /**
     * @var Book The book this review is about.
     *
     * @ORM\ManyToOne(targetEntity="Book", inversedBy="reviews")
     * @Assert\NotNull
     */
    private $book;

    // ...
}
```

After updating the entities by adding those `@Assert\*` annotations (as for Doctrine, you can use XML or YAML formats if you
prefer), try again the previous `POST` request.

    isbn: This value is neither a valid ISBN-10 nor a valid ISBN-13.
    title: This value should not be blank.

You now get proper validation error messages, always serialized using the Hydra error format (API Problem is also supported).
Those errors are easy to parse client-side. By adding the proper validation constraints, we also noticed that the provided
ISBN number isn't valid...

Here we are! We have created a working and very powerful hypermedia REST API in a few minutes, and by writing only a few
lines of PHP. But we only covered the basics.

## Other features

They are many more features to learn! Read [the full documentation](../core/index.md) to discover how to use them and how
to extend API Platform to fit your needs.
API Platform is incredibly efficient for prototyping and Rapid Application Development (RAD). But the framework is also
designed to create complex web APIs far beyond simple CRUD apps. It benefits from **strong extension points** and is **is
continuously optimized for performance.** It powers very high-traffic websites.

API Platform can also be extended using PHP libraries and Symfony bundles.

Here is a non-exhaustive list of popular API Platform extensions:

* Add [a user management system](../core/fosuser-bundle.md) (FOSUser)
* [Secure the API with JWT](https://github.com/lexik/LexikJWTAuthenticationBundle) (LexikJwtAuthenticationBundle) or [OAuth](https://github.com/FriendsOfSymfony/FOSOAuthServerBundle)
  (FosOAuthServer)
* [Add a Varnish reverse proxy and adopt a expiration or invalidation HTTP cache strategy](http://foshttpcachebundle.readthedocs.org)
  (FOSHttpCache)
* [Add CSRF protection when the API authentication relies on cookies](https://github.com/dunglas/DunglasAngularCsrfBundle)
  (DunglasAngularCsrfBundle)
* [Send mails](https://symfony.com/doc/current/cookbook/email/email.html) (Swift Mailer)
* [Execute async jobs and create micro-service architectures using RabbitMQ](https://github.com/php-amqplib/RabbitMqBundle)
  (RabbitMQBundle)

Keep in mind that you can use your favorite client-side technology: API Platform is tested and approved with React, Angular
1 & 2, Ionic and Swift but can work with any language able to send HTTP requests (even COBOL can do that).

To go further, the API Platform team maintains a demo application showing more advanced use cases like leveraging serialization
groups, user management or JWT and OAuth authentication. [Checkout the demo code source on GitHub](https://github.com/api-platform/demo)
and [browse it online](https://demo.api-platform.com).

Next chapter: [Testing And Specifying the API](testing.md)
