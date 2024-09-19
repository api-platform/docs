# Testing the API

Now that you have a functional API, you should write tests to ensure it has no bugs, and to prevent future regressions.
Some would argue that it's even better to [write tests first](https://martinfowler.com/bliki/TestDrivenDevelopment.html).

API Platform provides a set of helpful testing utilities to write unit tests, functional tests, and to create [test fixtures](https://en.wikipedia.org/wiki/Test_fixture#Software).

Let's learn how to use them!

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform-security/api-tests?cid=apip"><img src="/docs/symfony/images/symfonycasts-player.png" alt="Tests and Assertions screencast"><br>Watch the Tests & Assertions screencast</a></p>

In this article you'll learn how to use:

* [PHPUnit](https://phpunit.de), a testing framework to cover your classes with unit tests and to write
API-oriented functional tests thanks to its API Platform and [Symfony](https://symfony.com/doc/current/testing.html) integrations.
* [DoctrineFixturesBundle](https://symfony.com/bundles/DoctrineFixturesBundle/current/index.html), a bundle to load data fixtures in the database.
* [Foundry](https://github.com/zenstruck/foundry), an expressive fixtures generator to write data fixtures.

## Creating Data Fixtures

Before creating your functional tests, you will need a dataset to pre-populate your API and be able to test it.

First, install [Foundry](https://github.com/zenstruck/foundry) and [Doctrine/DoctrineFixturesBundle](https://github.com/doctrine/DoctrineFixturesBundle):

```console
docker compose exec php \
    composer require --dev foundry orm-fixtures
```

Thanks to Symfony Flex, [DoctrineFixturesBundle](https://github.com/doctrine/DoctrineFixturesBundle) and [Foundry](https://github.com/zenstruck/foundry) are ready to use!

Then, create some factories for [the bookstore API you created in the tutorial](index.md):

```console
docker compose exec php \
    bin/console make:factory 'App\Entity\Book'
docker compose exec php \
    bin/console make:factory 'App\Entity\Review'
```

Improve the default values:

```php
// src/Factory/BookFactory.php

    // ...

    protected function getDefaults(): array
    {
        return [
            'author' => self::faker()->name(),
            'description' => self::faker()->text(),
            'isbn' => self::faker()->isbn13(),
            'publication_date' => \DateTimeImmutable::createFromMutable(self::faker()->dateTime()),
            'title' => self::faker()->sentence(4),
        ];
    }
```

```php
// src/Factory/ReviewFactory.php
// ...
use function Zenstruck\Foundry\lazy;

    // ...

    protected function getDefaults(): array
    {
        return [
            'author' => self::faker()->name(),
            'body' => self::faker()->text(),
            'book' => lazy(fn() => BookFactory::randomOrCreate()),
            'publicationDate' => \DateTimeImmutable::createFromMutable(self::faker()->dateTime()),
            'rating' => self::faker()->numberBetween(0, 5),
        ];
    }
```

Create some stories:

```console
docker compose exec php \
    bin/console make:story 'DefaultBooks'
docker compose exec php \
    bin/console make:story 'DefaultReviews'
```

```php
// src/Story/DefaultBooksStory.php

namespace App\Story;

use App\Factory\BookFactory;
use Zenstruck\Foundry\Story;

final class DefaultBooksStory extends Story
{
    public function build(): void
    {
        BookFactory::createMany(100);
    }
}
```

```php
// src/Story/DefaultReviewsStory.php

namespace App\Story;

use App\Factory\ReviewFactory;
use Zenstruck\Foundry\Story;

final class DefaultReviewsStory extends Story
{
    public function build(): void
    {
        ReviewFactory::createMany(200);
    }
}
```

Edit your Fixtures:

```php
//src/DataFixtures/AppFixtures.php

namespace App\DataFixtures;

use App\Story\DefaultBooksStory;
use App\Story\DefaultReviewsStory;
use Doctrine\Bundle\FixturesBundle\Fixture;
use Doctrine\Persistence\ObjectManager;

class AppFixtures extends Fixture
{
    public function load(ObjectManager $manager): void
    {
        DefaultBooksStory::load();
        DefaultReviewsStory::load();
    }
}
```

You can now load your fixtures in the database with the following command:

```console
docker compose exec php \
    bin/console doctrine:fixtures:load
```

To learn more about fixtures, take a look at the documentation of [Foundry](https://symfony.com/bundles/ZenstruckFoundryBundle/current/index.html).
The list of available generators as well as a cookbook explaining how to create custom generators can be found in the documentation of [Faker](https://github.com/fakerphp/faker), the library used by Foundry under the hood.

## Writing Functional Tests

Now that you have some data fixtures for your API, you are ready to write functional tests with [PHPUnit](https://phpunit.de).

The API Platform test client implements the interfaces of the [Symfony HttpClient](https://symfony.com/doc/current/components/http_client.html). HttpClient is shipped with the API Platform distribution. The [Symfony test pack](https://github.com/symfony/test-pack/blob/main/composer.json), which includes PHPUnit as well as Symfony components useful for testing, is also included.

If you don't use the distribution, run `composer require --dev symfony/test-pack symfony/http-client` to install them.

Install [DAMADoctrineTestBundle](https://github.com/dmaicher/doctrine-test-bundle) to reset the database automatically before each test:

```console
docker compose exec php \
    composer require --dev dama/doctrine-test-bundle
```

And activate it in the `phpunit.xml.dist` file:

```xml
<!-- api/phpunit.xml.dist -->
<phpunit>
    <!-- ... -->

    <extensions>
        <extension class="DAMA\DoctrineTestBundle\PHPUnit\PHPUnitExtension"/>
    </extensions>
</phpunit>
```

Optionally, you can install [JSON Schema for PHP](https://github.com/justinrainbow/json-schema) if you want to use the [JSON Schema](https://json-schema.org) test assertions provided by API Platform:

```console
docker compose exec php \
    composer require --dev justinrainbow/json-schema
```

Your API is now ready to be functionally tested. Create your test classes under the `tests/` directory.

Here is an example of functional tests specifying the behavior of [the bookstore API you created in the tutorial](index.md):

```php
<?php
// api/tests/BooksTest.php

namespace App\Tests;

use ApiPlatform\Symfony\Bundle\Test\ApiTestCase;
use App\Entity\Book;
use App\Factory\BookFactory;
use Zenstruck\Foundry\Test\Factories;
use Zenstruck\Foundry\Test\ResetDatabase;

class BooksTest extends ApiTestCase
{
    // This trait provided by Foundry will take care of refreshing the database content to a known state before each test
    use ResetDatabase, Factories;

    public function testGetCollection(): void
    {
        // Create 100 books using our factory
        BookFactory::createMany(100);
    
        // The client implements Symfony HttpClient's `HttpClientInterface`, and the response `ResponseInterface`
        $response = static::createClient()->request('GET', '/books');

        $this->assertResponseIsSuccessful();
        // Asserts that the returned content type is JSON-LD (the default)
        $this->assertResponseHeaderSame('content-type', 'application/ld+json; charset=utf-8');

        // Asserts that the returned JSON is a superset of this one
        $this->assertJsonContains([
            '@context' => '/contexts/Book',
            '@id' => '/books',
            '@type' => 'hydra:Collection',
            'hydra:totalItems' => 100,
            'hydra:view' => [
                '@id' => '/books?page=1',
                '@type' => 'hydra:PartialCollectionView',
                'hydra:first' => '/books?page=1',
                'hydra:last' => '/books?page=4',
                'hydra:next' => '/books?page=2',
            ],
        ]);

        // Because test fixtures are automatically loaded between each test, you can assert on them
        $this->assertCount(30, $response->toArray()['hydra:member']);

        // Asserts that the returned JSON is validated by the JSON Schema generated for this resource by API Platform
        // This generated JSON Schema is also used in the OpenAPI spec!
        $this->assertMatchesResourceCollectionJsonSchema(Book::class);
    }

    public function testCreateBook(): void
    {
        $response = static::createClient()->request('POST', '/books', ['json' => [
            'isbn' => '0099740915',
            'title' => 'The Handmaid\'s Tale',
            'description' => 'Brilliantly conceived and executed, this powerful evocation of twenty-first century America gives full rein to Margaret Atwood\'s devastating irony, wit and astute perception.',
            'author' => 'Margaret Atwood',
            'publicationDate' => '1985-07-31T00:00:00+00:00',
        ]]);

        $this->assertResponseStatusCodeSame(201);
        $this->assertResponseHeaderSame('content-type', 'application/ld+json; charset=utf-8');
        $this->assertJsonContains([
            '@context' => '/contexts/Book',
            '@type' => 'Book',
            'isbn' => '0099740915',
            'title' => 'The Handmaid\'s Tale',
            'description' => 'Brilliantly conceived and executed, this powerful evocation of twenty-first century America gives full rein to Margaret Atwood\'s devastating irony, wit and astute perception.',
            'author' => 'Margaret Atwood',
            'publicationDate' => '1985-07-31T00:00:00+00:00',
            'reviews' => [],
        ]);
        $this->assertMatchesRegularExpression('~^/books/\d+$~', $response->toArray()['@id']);
        $this->assertMatchesResourceItemJsonSchema(Book::class);
    }

    public function testCreateInvalidBook(): void
    {
        static::createClient()->request('POST', '/books', ['json' => [
            'isbn' => 'invalid',
        ]]);

        $this->assertResponseStatusCodeSame(422);
        $this->assertResponseHeaderSame('content-type', 'application/ld+json; charset=utf-8');

        $this->assertJsonContains([
            '@context' => '/contexts/ConstraintViolationList',
            '@type' => 'ConstraintViolationList',
            'hydra:title' => 'An error occurred',
            'hydra:description' => 'isbn: This value is neither a valid ISBN-10 nor a valid ISBN-13.
title: This value should not be blank.
description: This value should not be blank.
author: This value should not be blank.
publicationDate: This value should not be null.',
        ]);
    }

    public function testUpdateBook(): void
    {
        // Only create the book we need with a given ISBN
        BookFactory::createOne(['isbn' => '9781344037075']);
    
        $client = static::createClient();
        // findIriBy allows to retrieve the IRI of an item by searching for some of its properties.
        $iri = $this->findIriBy(Book::class, ['isbn' => '9781344037075']);

        // Use the PATCH method here to do a partial update
        $client->request('PATCH', $iri, [
            'json' => [
                'title' => 'updated title',
            ],
            'headers' => [
                'Content-Type' => 'application/merge-patch+json',
            ]           
        ]);

        $this->assertResponseIsSuccessful();
        $this->assertJsonContains([
            '@id' => $iri,
            'isbn' => '9781344037075',
            'title' => 'updated title',
        ]);
    }

    public function testDeleteBook(): void
    {
        // Only create the book we need with a given ISBN
        BookFactory::createOne(['isbn' => '9781344037075']);

        $client = static::createClient();
        $iri = $this->findIriBy(Book::class, ['isbn' => '9781344037075']);

        $client->request('DELETE', $iri);

        $this->assertResponseStatusCodeSame(204);
        $this->assertNull(
            // Through the container, you can access all your services from the tests, including the ORM, the mailer, remote API clients...
            static::getContainer()->get('doctrine')->getRepository(Book::class)->findOneBy(['isbn' => '9781344037075'])
        );
    }
}
```

As you can see, the example uses the [trait `ResetDatabase`](https://symfony.com/bundles/ZenstruckFoundryBundle/current/index.html#database-reset)
from [Foundry](https://github.com/zenstruck/foundry) which will, at the beginning of each
test, purge the database, begin a transaction, and, at the end of each test, roll back the
transaction previously begun. Because of this, you can run your tests without worrying about fixtures.

There is one caveat though: in some tests, it is necessary to perform multiple requests in one test, for example when creating a user via the API and checking that a subsequent login using the same password works. However, the client will by default reboot the kernel, which will reset the database. You can prevent this by adding `$client->disableReboot();` to such tests.

All you have to do now is to run your tests:

```console
docker compose exec php \
    bin/phpunit
```

If everything is working properly, you should see `OK (5 tests, 17 assertions)`.
Your REST API is now properly tested!

Check out the [testing documentation](../core/testing.md) to discover the full range of assertions and other features provided by API Platform's test utilities.

## Writing Unit Tests

In addition to integration tests written using the helpers provided by `ApiTestCase`, all the classes of your project should be covered by [unit tests](https://en.wikipedia.org/wiki/Unit_testing).
To do so, learn how to write unit tests with [PHPUnit](https://phpunit.de/) and [its Symfony/API Platform integration](https://symfony.com/doc/current/testing.html).

## Continuous Integration, Continuous Delivery and Continuous Deployment

Running your test suite in your [CI/CD pipeline](https://en.wikipedia.org/wiki/Continuous_integration) is important to ensure good quality and delivery time.

The API Platform distribution is [shipped with a GitHub Actions workflow](https://github.com/api-platform/api-platform/blob/main/.github/workflows/ci.yml) that builds the Docker images, does a [smoke test](https://en.wikipedia.org/wiki/Smoke_testing_(software)) to check that the application's entrypoint is accessible, and runs PHPUnit.

The API Platform Demo [contains a CD workflow](https://github.com/api-platform/demo/tree/main/.github/workflows) that uses [the Helm chart provided with the distribution](../deployment/kubernetes.md) to deploy the app on a Kubernetes cluster.

## Additional and Alternative Testing Tools

You may also be interested in these alternative testing tools (not included in the API Platform distribution):

* [Hoppscotch](https://docs.hoppscotch.io/features/tests), create functional test for your API
* [Hoppscotch](https://docs.hoppscotch.io/documentation/features/rest-api-testing/), create functional test for your API
  Platform project using a nice UI, benefit from its Swagger integration and run tests in the CI using [the command-line tool](https://docs.hoppscotch.io/cli);
* [Behat](https://behat.org), a
  [behavior-driven development (BDD)](https://en.wikipedia.org/wiki/Behavior-driven_development) framework to write the API
  specification as user stories and in natural language then execute these scenarios against the application to validate
  its behavior;
* [Blackfire Player](https://blackfire.io/player), a nice DSL to crawl HTTP services, assert responses, and extract data
  from HTML/XML/JSON responses;
* [PHP Matcher](https://github.com/coduo/php-matcher), the Swiss Army knife of JSON document testing.

## Using the API Platform Distribution for End-to-End Testing

If you would like to verify that your stack (including services such as the DBMS, web server, [Varnish](https://varnish-cache.org/))
works, you need [end-to-end (E2E) testing](https://wiki.c2.com/?EndToEndPrinciple). To do so, we recommend using [Playwright](https://playwright.dev) if you use have PWA/JavaScript-heavy app, or [Symfony Panther](https://github.com/symfony/panther) if you mostly use Twig.

Usually, E2E testing should be done with a production-like setup. For your convenience, you may [run our Docker Compose setup
for production locally](../deployment/docker-compose.md#running-the-docker-compose-setup-for-production-locally).
