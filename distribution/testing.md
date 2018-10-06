# Testing and Specifying the API

A set of useful tools to specify and test your API are easily installable in the API Platform distribution:

* [PHPUnit](https://phpunit.de/) allows to cover your classes with unit tests and to write functional tests thanks to his
  Symfony integration.
* [Behat](http://docs.behat.org/) (a [Behavior-driven development](http://en.wikipedia.org/wiki/Behavior-driven_development)
  framework) and its [Behatch extension](https://github.com/Behatch/contexts) (a set of contexts dedicated to REST API and
  JSON documents) are convenient to specify and test your API: write the API specification as user stories and in natural
  language then execute these scenarios against the application to validate its behavior.

Take a look at [the Symfony documentation about testing](https://symfony.com/doc/current/testing.html) to learn how to use
PHPUnit in your API Platform project.

Installing Behat is easy enough following these steps:

    $ docker-compose exec php composer require --dev behat/behat
    $ docker-compose exec php vendor/bin/behat -V
    $ docker-compose exec php vendor/bin/behat --init

This will install Behat in your project and creates a directory `features` where you can place your feature file(s).

Here is an example of a [Gherkin](http://docs.behat.org/en/latest/user_guide/gherkin.html) feature file specifying the behavior
of [the bookstore API we created in the tutorial](index.md). Thanks to Behatch, this feature file can be executed against
the API without having to write a single line of PHP.

```gherkin
# features/books.feature
Feature: Manage books and their reviews
  In order to manage books and their reviews
  As a client software developer
  I need to be able to retrieve, create, update and delete them through the API.

  # the "@createSchema" annotation provided by API Platform creates a temporary SQLite database for testing the API
  @createSchema
  Scenario: Create a book
    When I add "Content-Type" header equal to "application/ld+json"
    And I add "Accept" header equal to "application/ld+json"
    And I send a "POST" request to "/books" with body:
    """
    {
      "isbn": "9781782164104",
      "title": "Persistence in PHP with the Doctrine ORM",
      "description": "This book is designed for PHP developers and architects who want to modernize their skills through better understanding of Persistence and ORM.",
      "author": "Kévin Dunglas",
      "publicationDate": "2013-12-01"
    }
    """
    Then the response status code should be 201
    And the response should be in JSON
    And the header "Content-Type" should be equal to "application/ld+json; charset=utf-8"
    And the JSON should be equal to:
    """
    {
      "@context": "/contexts/Book",
      "@id": "/books/1",
      "@type": "Book",
      "id": 1,
      "isbn": "9781782164104",
      "title": "Persistence in PHP with the Doctrine ORM",
      "description": "This book is designed for PHP developers and architects who want to modernize their skills through better understanding of Persistence and ORM.",
      "author": "K\u00e9vin Dunglas",
      "publicationDate": "2013-12-01T00:00:00+00:00",
      "reviews": []
    }
    """

  Scenario: Retrieve the book list
    When I add "Accept" header equal to "application/ld+json"
    And I send a "GET" request to "/books"
    Then the response status code should be 200
    And the response should be in JSON
    And the header "Content-Type" should be equal to "application/ld+json; charset=utf-8"
    And the JSON should be equal to:
    """
    {
      "@context": "/contexts/Book",
      "@id": "/books",
      "@type": "hydra:Collection",
      "hydra:member": [
        {
          "@id": "/books/1",
          "@type": "Book",
          "id": 1,
          "isbn": "9781782164104",
          "title": "Persistence in PHP with the Doctrine ORM",
          "description": "This book is designed for PHP developers and architects who want to modernize their skills through better understanding of Persistence and ORM.",
          "author": "K\u00e9vin Dunglas",
          "publicationDate": "2013-12-01T00:00:00+00:00",
          "reviews": []
        }
      ],
      "hydra:totalItems": 1
    }
    """

  Scenario: Throw errors when a post is invalid
    When I add "Content-Type" header equal to "application/ld+json"
    And I add "Accept" header equal to "application/ld+json"
    And I send a "POST" request to "/books" with body:
    """
    {
      "isbn": "1312",
      "title": "",
      "description": "Yo!",
      "author": "Me!",
      "publicationDate": "2016-01-01"
    }
    """
    Then the response status code should be 400
    And the response should be in JSON
    And the header "Content-Type" should be equal to "application/ld+json; charset=utf-8"
    And the JSON should be equal to:
    """
    {
      "@context": "/contexts/ConstraintViolationList",
      "@type": "ConstraintViolationList",
      "hydra:title": "An error occurred",
      "hydra:description": "isbn: This value is neither a valid ISBN-10 nor a valid ISBN-13.\ntitle: This value should not be blank.",
      "violations": [
        {
          "propertyPath": "isbn",
          "message": "This value is neither a valid ISBN-10 nor a valid ISBN-13."
        },
        {
          "propertyPath": "title",
          "message": "This value should not be blank."
        }
      ]
    }
    """

  # The "@dropSchema" annotation must be added on the last scenario of the feature file to drop the temporary SQLite database
  @dropSchema
    Scenario: Add a review
    When I add "Content-Type" header equal to "application/ld+json"
    When I add "Accept" header equal to "application/ld+json"
    And I send a "POST" request to "/reviews" with body:
    """
    {
      "rating": 5,
      "body": "Must have!",
      "author": "Foo Bar",
      "publicationDate": "2016-01-01",
      "book": "/books/1"
    }
    """
    Then the response status code should be 201
    And the response should be in JSON
    And the header "Content-Type" should be equal to "application/ld+json; charset=utf-8"
    And the JSON should be equal to:
    """
    {
      "@context": "/contexts/Review",
      "@id": "/reviews/1",
      "@type": "Review",
      "id": 1,
      "rating": 5,
      "body": "Must have!",
      "author": "Foo Bar",
      "publicationDate": "2016-01-01T00:00:00+00:00",
      "book": "/books/1"
    }
    """
```

The API Platform flavor of Behat also comes with a temporary SQLite database dedicated to tests. It works out of the box.

Clear the cache of the `test` environment:

    $ docker-compose exec php bin/console cache:clear --env=test

Then run:

    $ docker-compose exec php vendor/bin/behat

Everything should be green now. Your Linked Data API is now specified and tested thanks to Behat!

You may also be interested in these alternative testing tools (not included in the API Platform distribution):

* [Postman tests](https://www.getpostman.com/docs/writing_tests) (proprietary): create functional test for your API Platform project
  using a nice UI, benefit from [the Swagger integration](https://www.getpostman.com/docs/importing_swagger) and run tests
  test in the CI using [newman](https://github.com/postmanlabs/newman).
* [PHP Matcher](https://github.com/coduo/php-matcher): the Swiss Army knife of JSON document testing.

## Running Unit Tests with PHPUnit

To run your [PHPUnit](https://phpunit.de/) test suite, execute the following command:

    $ docker-compose exec php vendor/bin/phpunit
