# Testing Utilities

API Platform Core provides a set of useful utilities dedicated to API testing.
For an overview of how to test an API Platform app, be sure to read [the testing cookbook first](../distribution/testing.md).

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform-security/api-tests?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Test and Assertions screencast"><br>Watch the API Tests & Assertions screencast</a></p>

## The Test HttpClient

API Platform provides its own implementation of the [Symfony HttpClient](https://symfony.com/doc/current/components/http_client.html)'s interfaces, tailored to be used directly in [PHPUnit](https://phpunit.de/) test classes.

While all the convenient features of Symfony HttpClient are available and usable directly, under the hood the API Platform implementation manipulates [the Symfony HttpKernel](https://symfony.com/doc/current/components/http_kernel.html) directly to simulate HTTP requests and responses.
This approach results in a huge performance boost compared to triggering real network requests.
It also allows access to the [Symfony HttpKernel](https://symfony.com/doc/current/components/http_kernel.html) and to all your services via the [Dependency Injection Container](https://symfony.com/doc/current/testing.html#accessing-the-container). 
Reuse them to run, for instance, SQL queries or requests to external APIs directly from your tests.

Install the `symfony/http-client` and `symfony/browser-kit` packages to enabled the API Platform test client:

    $ docker-compose exec php composer require symfony/http-client symfony/browser-kit

To use the testing client, your test class must extend the `ApiTestCase` class:

```php
<?php
// api/tests/BooksTest.php

namespace App\Tests;

use ApiPlatform\Core\Bridge\Symfony\Bundle\Test\ApiTestCase;

class BooksTest extends ApiTestCase
{
    public function testGetCollection(): void
    {
        $response = static::createClient()->request('GET', '/books');
        // your assertions here...
    }
}
```

Refer to [the Symfony HttpClient documentation](https://symfony.com/doc/current/components/http_client.html) to discover all the features of the client (custom headers, JSON encoding and decoding, HTTP Basic and Bearer authentication and cookies support, among other things).

Note that you can create your own test case class extending the ApiTestCase. For example to set up a Json Web Token authentification:

```php
<?php

namespace App\Tests;

use ApiPlatform\Core\Bridge\Symfony\Bundle\Test\ApiTestCase;
use ApiPlatform\Core\Bridge\Symfony\Bundle\Test\Client;
use Hautelook\AliceBundle\PhpUnit\RefreshDatabaseTrait;

abstract class AbstractTest extends ApiTestCase
{
    private $token;
    private $clientWithCredentials;

    use RefreshDatabaseTrait;

    public function setUp(): void
    {
        self::bootKernel();
    }

    protected function createClientWithCredentials($token = null): Client
    {
        $token = $token ?: $this->getToken();

        return static::createClient([], ['headers' => ['authorization' => 'Bearer '.$token]]);
    }

    /**
     * Use other credentials if needed.
     */
    protected function getToken($body = []): string
    {
        if ($this->token) {
            return $this->token;
        }

        $response = static::createClient()->request('POST', '/login', ['body' => $body ?: [
            'username' => 'admin@example.com',
            'password' => '$3cr3t',
        ]]);

        $this->assertResponseIsSuccessful();
        $data = json_decode($response->getContent());
        $this->token = $data->access_token;

        return $data->access_token;
    }
}
```

Use it by extending the `AbstractTest` class. For example this class tests the `/users` resource accessibility where only the admin can retrieve the collection:

```php
<?php
namespace App\Tests;

final class UsersTest extends AbstractTest
{
    public function testAdminResource()
    {
        $response = $this->createClientWithCredentials()->request('GET', '/users');
        $this->assertResponseIsSuccessful();
    }

    public function testLoginAsUser()
    {
        $token = $this->getToken([
            'username' => 'user@example.com',
            'password' => '$3cr3t',
        ]);

        $response = $this->createClientWithCredentials($token)->request('GET', '/users');
        $this->assertJsonContains(['hydra:description' => 'Access Denied.']);
        $this->assertResponseStatusCodeSame('403');
    }
}
```

## API Test Assertions

In addition to [the built-in ones](https://phpunit.readthedocs.io/en/latest/assertions.html), API Platform provides convenient PHPUnit assertions dedicated to API testing:

```php
<?php
// api/tests/MyTest.php

namespace App\Tests;

use ApiPlatform\Core\Bridge\Symfony\Bundle\Test\ApiTestCase;

class MyTest extends ApiTestCase
{
    public function testSomething(): void
    {
        // static::createClient()->request(...);

        // Asserts that the returned JSON is equal to the passed one
        $this->assertJsonEquals(/* a JSON document as an array or as a string */);

        // Asserts that the returned JSON is a superset of the passed one
        $this->assertJsonContains(/* a JSON document as an array or as a string */);

        // justinrainbow/json-schema must be installed to use the following assertions

        // Asserts that the returned JSON matches the passed JSON Schema
        $this->assertMatchesJsonSchema(/* a JSON Schema as an array or as a string */);

        // Asserts that the returned JSON is validated by the JSON Schema generated for this resource by API Platform

        // For collections
        $this->assertMatchesResourceCollectionJsonSchema(YourApiResource::class);
        // And for items
        $this->assertMatchesResourceItemJsonSchema(YourApiResource::class);
    }
}
```

There is a also a method to find the IRI matching a given resource and some criterias:

```php
<?php
// api/tests/BooksTest.php

namespace App\Tests;

use ApiPlatform\Core\Bridge\Symfony\Bundle\Test\ApiTestCase;

class BooksTest extends ApiTestCase
{
    public function testFindBook(): void
    {
        // Asserts that the returned JSON is equal to the passed one
        $iri = $this->findIriBy(Book::class, ['isbn' => '9780451524935']);
        static::createClient()->request('GET', $iri);
        $this->assertResponseIsSuccessful();
    }
}
```

## HTTP Test Assertions

All tests assertions provided by Symfony (assertions for status codes, headers, cookies, XML documents...) can be used out of the box with the API Platform test client:

```php
<?php
// api/tests/BooksTest.php

namespace App\Tests;

use ApiPlatform\Core\Bridge\Symfony\Bundle\Test\ApiTestCase;

class BooksTest extends ApiTestCase
{
    public function testGetCollection(): void
    {
        static::createClient()->request('GET', '/books');

        $this->assertResponseIsSuccessful();
        $this->assertResponseHeaderSame('content-type', 'application/ld+json; charset=utf-8');
    }
}
```

[Check out the dedicated Symfony documentation entry](https://symfony.com/doc/current/testing/functional_tests_assertions.html).
