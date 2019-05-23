# Testing and Specifying the API

Now that you have a functional API, it might be interesting to write some tests to ensure your API have no potential
bugs. A set of useful tools to specify and test your API are easily installable in the API Platform distribution. We
recommend you and we will focus on two tools:

* [Alice](https://github.com/nelmio/alice), an expressive fixtures generator to write data fixtures, and its Symfony
integration, [AliceBundle](https://github.com/hautelook/AliceBundle#database-testing);
* [PHPUnit](https://phpunit.de/index.html), a testing framework to cover your classes with unit tests and to write
functional tests thanks to its Symfony integration, [PHPUnit Bridge](https://symfony.com/doc/current/components/phpunit_bridge.html).

Official Symfony recipes are provided for both tools.

## Creating Data Fixtures

Before creating your functional tests, you will need a dataset to pre-populate your API and be able to test it.

First, install [Alice](https://github.com/nelmio/alice) and [AliceBundle](https://github.com/hautelook/AliceBundle):

    $ docker-compose exec php composer require --dev alice

Thanks to Symfony Flex, [AliceBundle](https://github.com/hautelook/AliceBundle/blob/master/README.md) is ready to use
and you can place your data fixtures files in a directory named `fixtures/`.

Then, create some fixtures for [the bookstore API you created in the tutorial](index.md):

```yaml
# api/fixtures/book.yaml

App\Entity\Book:
    book_{1..10}:
        isbn: <isbn13()>
        title: <sentence(4)>
        description: <text()>
        author: <name()>
        publicationDate: <dateTime()>
```

```yaml
# api/fixtures/review.yaml

App\Entity\Review:
    review_{1..20}:
        rating: <numberBetween(0, 5)>
        body: <text()>
        author: <name()>
        publicationDate: <dateTime()>
        book: '@book_*'
```

You can now load your fixtures in the database with the following command:

    $ docker-compose exec php bin/console hautelook:fixtures:load

To learn more about fixtures, take a look at the documentation of [Alice](https://github.com/nelmio/alice/blob/master/README.md#table-of-contents)
and [AliceBundle](https://github.com/hautelook/AliceBundle/blob/master/README.md).

## Writing Functional Tests

Now that you have some data fixtures for your API, you are ready to write functional tests with [PHPUnit](https://phpunit.de/index.html).

Install the Symfony test pack which includes [PHPUnit Bridge](https://symfony.com/doc/current/components/phpunit_bridge.html):

    $ docker-compose exec php composer require --dev test-pack

Your API is ready to be functionally tested. Create your test classes under the `tests/` directory.

Here is an example of functional tests specifying the behavior of [the bookstore API you created in the tutorial](index.md):

```php
<?php
// api/tests/ApiTest.php

namespace App\Tests;

use App\Entity\Book;
use Hautelook\AliceBundle\PhpUnit\RefreshDatabaseTrait;
use Symfony\Bundle\FrameworkBundle\Client;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
use Symfony\Component\HttpFoundation\Response;

class ApiTest extends WebTestCase
{
    use RefreshDatabaseTrait;

    /** @var Client */
    protected $client;

    /**
     * Retrieves the book list.
     */
    public function testRetrieveTheBookList(): void
    {
        $response = $this->request('GET', '/books');
        $json = json_decode($response->getContent(), true);

        $this->assertEquals(200, $response->getStatusCode());
        $this->assertEquals('application/ld+json; charset=utf-8', $response->headers->get('Content-Type'));

        $this->assertArrayHasKey('hydra:totalItems', $json);
        $this->assertEquals(10, $json['hydra:totalItems']);

        $this->assertArrayHasKey('hydra:member', $json);
        $this->assertCount(10, $json['hydra:member']);
    }

    /**
     * Throws errors when data are invalid.
     */
    public function testThrowErrorsWhenDataAreInvalid(): void
    {
        $data = [
            'isbn' => '1312',
            'title' => '',
            'author' => 'Kévin Dunglas',
            'description' => 'This book is designed for PHP developers and architects who want to modernize their skills through better understanding of Persistence and ORM.',
            'publicationDate' => '2013-12-01',
        ];

        $response = $this->request('POST', '/books', $data);
        $json = json_decode($response->getContent(), true);

        $this->assertEquals(400, $response->getStatusCode());
        $this->assertEquals('application/ld+json; charset=utf-8', $response->headers->get('Content-Type'));

        $this->assertArrayHasKey('violations', $json);
        $this->assertCount(2, $json['violations']);

        $this->assertArrayHasKey('propertyPath', $json['violations'][0]);
        $this->assertEquals('isbn', $json['violations'][0]['propertyPath']);

        $this->assertArrayHasKey('propertyPath', $json['violations'][1]);
        $this->assertEquals('title', $json['violations'][1]['propertyPath']);
    }

    /**
     * Creates a book.
     */
    public function testCreateABook(): void
    {
        $data = [
            'isbn' => '9781782164104',
            'title' => 'Persistence in PHP with Doctrine ORM',
            'description' => 'This book is designed for PHP developers and architects who want to modernize their skills through better understanding of Persistence and ORM. You\'ll learn through explanations and code samples, all tied to the full development of a web application.',
            'author' => 'Kévin Dunglas',
            'publicationDate' => '2013-12-01',
        ];

        $response = $this->request('POST', '/books', $data);
        $json = json_decode($response->getContent(), true);

        $this->assertEquals(201, $response->getStatusCode());
        $this->assertEquals('application/ld+json; charset=utf-8', $response->headers->get('Content-Type'));

        $this->assertArrayHasKey('isbn', $json);
        $this->assertEquals('9781782164104', $json['isbn']);
    }

    /**
     * Updates a book.
     */
    public function testUpdateABook(): void
    {
        $data = [
            'isbn' => '9781234567897',
        ];

        $response = $this->request('PUT', $this->findOneIriBy(Book::class, ['isbn' => '9790456981541']), $data);
        $json = json_decode($response->getContent(), true);

        $this->assertEquals(200, $response->getStatusCode());
        $this->assertEquals('application/ld+json; charset=utf-8', $response->headers->get('Content-Type'));

        $this->assertArrayHasKey('isbn', $json);
        $this->assertEquals('9781234567897', $json['isbn']);
    }

    /**
     * Deletes a book.
     */
    public function testDeleteABook(): void
    {
        $response = $this->request('DELETE', $this->findOneIriBy(Book::class, ['isbn' => '9790456981541']));

        $this->assertEquals(204, $response->getStatusCode());

        $this->assertEmpty($response->getContent());
    }

    /**
     * Retrieves the documentation.
     */
    public function testRetrieveTheDocumentation(): void
    {
        $response = $this->request('GET', '/', null, ['Accept' => 'text/html']);

        $this->assertEquals(200, $response->getStatusCode());
        $this->assertEquals('text/html; charset=UTF-8', $response->headers->get('Content-Type'));

        $this->assertContains('Hello API Platform', $response->getContent());
    }

    protected function setUp()
    {
        parent::setUp();

        $this->client = static::createClient();
    }

    /**
     * @param string|array|null $content
     */
    protected function request(string $method, string $uri, $content = null, array $headers = []): Response
    {
        $server = ['CONTENT_TYPE' => 'application/ld+json', 'HTTP_ACCEPT' => 'application/ld+json'];
        foreach ($headers as $key => $value) {
            if (strtolower($key) === 'content-type') {
                $server['CONTENT_TYPE'] = $value;

                continue;
            }

            $server['HTTP_'.strtoupper(str_replace('-', '_', $key))] = $value;
        }

        if (is_array($content) && false !== preg_match('#^application/(?:.+\+)?json$#', $server['CONTENT_TYPE'])) {
            $content = json_encode($content);
        }

        $this->client->request($method, $uri, [], [], $server, $content);

        return $this->client->getResponse();
    }

    protected function findOneIriBy(string $resourceClass, array $criteria): string
    {
        $resource = static::$container->get('doctrine')->getRepository($resourceClass)->findOneBy($criteria);

        return static::$container->get('api_platform.iri_converter')->getIriFromitem($resource);
    }
}
```

As you can see, the example uses the [trait `RefreshDatabaseTrait`](https://github.com/hautelook/AliceBundle#database-testing)
from [AliceBundle](https://github.com/hautelook/AliceBundle/blob/master/README.md) which will, at the beginning of each
test, purge the database, load fixtures, begin a transaction, and, at the end of each test, roll back the
transaction previously begun. Because of this, you can run your tests without worrying about fixtures.

All you have to do now is to run your tests:

    $ docker-compose exec php bin/phpunit

If everything is working properly, you should see `OK (6 tests, 27 assertions)`. Your Linked Data API is now specified
and tested thanks to [PHPUnit](https://phpunit.de/index.html)!

### Additional and Alternative Testing Tools

You may also be interested in these alternative testing tools (not included in the API Platform distribution):

* [ApiTestCase](https://github.com/lchrusciel/ApiTestCase), a handy [PHPUnit](https://phpunit.de/index.html) test case
  for going further by testing JSON and XML APIs in your Symfony applications;
* [Behat](http://behat.org/en/latest/) and its [Behatch extension](https://github.com/Behatch/contexts), a
  [Behavior-Driven development](https://en.wikipedia.org/wiki/Behavior-driven_development) framework to write the API
  specification as user stories and in natural language then execute these scenarios against the application to validate
  its behavior;
* [Blackfire Player](https://blackfire.io/player), a nice DSL to crawl HTTP services, assert responses, and extract data
  from HTML/XML/JSON responses ([see example in API Platform Demo](https://github.com/api-platform/demo/blob/master/test-api.bkf));
* [Postman tests](https://www.getpostman.com/docs/writing_tests) (proprietary), create functional test for your API
  Platform project using a nice UI, benefit from [the Swagger integration](https://www.getpostman.com/docs/importing_swagger)
  and run tests in the CI using [newman](https://github.com/postmanlabs/newman);
* [PHP Matcher](https://github.com/coduo/php-matcher), the Swiss Army knife of JSON document testing.

## Writing Unit Tests

Take a look at [the Symfony documentation about testing](https://symfony.com/doc/current/testing.html) to learn how to
write unit tests with [PHPUnit](https://phpunit.de/index.html) in your API Platform project.
