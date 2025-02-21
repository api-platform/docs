# Testing the API with Laravel

For an introduction to testing using API Platform, refer to the [Core Testing Documentation](../core/testing.md), or access the
[Symfony Testing Guide](../symfony/testing.md).

Let's learn how to use tests with Laravel!

In this article, you'll learn how to use:

- **[Pest](https://pestphp.com/)**: A testing framework that enables you to write unit tests for your classes and create
API-oriented functional tests, thanks to its integrations with API Platform and [Laravel](https://laravel.com/docs/testing).
- **[PHPUnit](https://phpunit.de)**: A testing framework for writing unit tests for your classes and conducting API-oriented
functional tests, with support for API Platform and [Laravel](https://laravel.com/docs/testing).

> [!TIP]
> Pest is built on top of PHPUnit and introduces additional features along with a syntax inspired by Ruby's RSpec and the
> Jest testing APIs.

## Tests with Pest

> [!TIP]
> Even if you are using Pest, you can also use PHPUnit's assertion API, which can be useful if you're already familiar
> with PHPUnit's assertion API or if you need to perform more complex assertions that aren't available in Pest's expectation API.
> For more information see the [Pest Assertion API](https://pestphp.com/docs/writing-tests#content-assertion-api) documentation.

### Installing Pest

By default, when using Laravel, Pest is pre-configured through the Composer plugin `pestphp/pest-plugin`. You can find this plugin listed in the `allow-plugins` section of your `composer.json` file.

To check the Pest installation, run the following command:

```console
php artisan test
```

If for some reason, Pest is not installed refer to the [Pest Installation Guide](https://pestphp.com/docs/installation).

In that case, you can run Pest using:

```console
./vendor/bin/pest
```

### Writing Functional Tests with Pest

#### Generate the Factory

Using Laravel, you can efficiently test databases by combining seeding with model factories. Model factories allow you
to generate large amounts of test data quickly, while seeding ensures your database is pre-populated with the necessary records.

To create a factory for your model, you can use [Laravel Artisan](https://laravel.com/docs/artisan) command.
For example, to create a factory for a Book model, run:

```console
php artisan make:factory BookFactory
```

For advanced customization and configuration, refer to the [Defining model Factories Laravel Guide](https://laravel.com/docs/eloquent-factories#defining-model-factories).

Then, you can now use your factory in tests to quickly generate model instances.

#### Writing Pest tests

Here’s an example of tests, which use the Factory:

```php
<?php

use App\Models\Book;
use Illuminate\Foundation\Testing\RefreshDatabase;
use ApiPlatform\Laravel\Test\ApiTestAssertionsTrait;

uses(RefreshDatabase::class, ApiTestAssertionsTrait::class);

it('fetches the collection of books')
    ->test(function () {
        // Create 100 books using the factory
        Book::factory()->count(100)->create();

        // Send a GET request to the collection endpoint
        $response = $this->getJson('/api/books');

        // Assert that the response is successful (200 OK)
        $response->assertStatus(200);

        // Check the Content-Type header
        $response->assertHeader('Content-Type', 'application/json');

        // Assert the returned JSON contains the expected structure using assertJsonContains from the trait
        $this->assertJsonContains([
            '@context' => '/contexts/Book',
            '@id' => '/books',
            '@type' => 'Collection',
            'totalItems' => 100,
            'view' => [
                '@id' => '/books?page=1',
                '@type' => 'PartialCollectionView',
                'first' => '/books?page=1',
                'last' => '/books?page=4',
                'next' => '/books?page=2',
            ],
        ], $response->json());

        // Assert that 30 items are returned in the response
        $this->assertCount(30, $response->json('data'));
    });

it('creates a valid book')
    ->test(function () {
        // Send a POST request to create a book
        $response = $this->postJson('/api/books', [
            'isbn' => '0099740915',
            'title' => 'The Handmaid\'s Tale',
            'description' => 'Brilliantly conceived and executed, this powerful evocation of twenty-first century America...',
            'author' => 'Margaret Atwood',
            'publication_date' => '1985-07-31',
        ]);

        // Assert that the book was created successfully (201)
        $response->assertStatus(201);

        // Check the Content-Type header
        $response->assertHeader('Content-Type', 'application/json');

        // Assert the returned JSON contains the expected book information using assertJsonContains
        $this->assertJsonContains([
            '@context' => '/contexts/Book',
            '@type' => 'Book',
            'isbn' => '0099740915',
            'title' => 'The Handmaid\'s Tale',
            'author' => 'Margaret Atwood',
            'publication_date' => '1985-07-31',
            'reviews' => [],
        ], $response->json());

        // Assert that the URI of the created resource matches the expected format
        $this->assertMatchesRegularExpression('~^/api/books/\d+$~', $response->json('@id'));
    });

it('creates an invalid book and validates error handling')
    ->test(function () {
        // Send a POST request with invalid data
        $response = $this->postJson('/api/books', [
            'isbn' => 'invalid',
        ]);

        // Assert that the response status is 422 Unprocessable Entity
        $response->assertStatus(422);

        // Check the Content-Type header
        $response->assertHeader('Content-Type', 'application/json');

        // Assert the JSON response contains the validation errors using assertJsonContains
        $this->assertJsonContains([
            '@context' => '/contexts/ConstraintViolationList',
            '@type' => 'ConstraintViolationList',
            'title' => 'An error occurred',
            'description' => [
                'isbn' => 'This value is neither a valid ISBN-10 nor a valid ISBN-13.',
                'title' => 'This value should not be blank.',
                'description' => 'This value should not be blank.',
                'author' => 'This value should not be blank.',
                'publication_date' => 'This value should not be null.',
            ],
        ], $response->json());
    });

it('updates a book')
    ->test(function () {
        // Create a book using the factory
        $book = Book::factory()->create(['isbn' => '9781344037075']);

        // Get the IRI of the book using getIriFromResource from the trait
        $iri = $this->getIriFromResource($book);

        // Send a PATCH request to update the book's title
        $response = $this->patchJson($iri, [
            'title' => 'Updated Title',
        ]);

        // Assert that the response is successful (200 OK)
        $response->assertStatus(200);

        // Assert the JSON response contains the updated book information using assertJsonContains
        $this->assertJsonContains([
            '@id' => $iri,
            'isbn' => '9781344037075',
            'title' => 'Updated Title',
        ], $response->json());
    });

it('deletes a book')
    ->test(function () {
        // Create a book using the factory
        $book = Book::factory()->create(['isbn' => '9781344037075']);

        // Get the IRI of the book using getIriFromResource from the trait
        $iri = $this->getIriFromResource($book);

        // Send a DELETE request to remove the book
        $response = $this->deleteJson($iri);

        // Assert that the response status is 204 No Content
        $response->assertStatus(204);

        // Assert that the book is no longer in the database
        $this->assertDatabaseMissing('books', ['id' => $book->id]);
    });
```

In the example above, the [RefreshDatabase Trait](https://laravel.com/docs/database-testing#resetting-the-database-after-each-test)
is used to ensure that the database is automatically reset between test runs. This guarantees that each test starts with
a clean database state, avoiding conflicts from residual data and ensuring test isolation.

This trait is especially useful when testing operations that modify the database, as it rolls back any changes made during the test.
As a result, your test environment remains reliable and consistent across multiple test executions.

#### Run Pest tests

If everything is working properly, you should see `Tests: 5 passed (15 assertions)`.
Your REST API is now properly tested!

Check out the [API Test Assertions section](./#api-test-assertions-with-laravel) to discover the full range of assertions
and other features provided by API Platform's test utilities.

### Migrating from PHPUnit to Pest

If you want to migrate from PHPUnit to Pest, refer to [Migrating from PHPUnit Guide](https://pestphp.com/docs/migrating-from-phpunit-guide)
and [Installation Guide](https://pestphp.com/docs/installation).

## Tests with PHPUnit

### Installing PHPUnit

By default, with Laravel, PHPUnit is already a dependency in your project. You may see `phpunit/phpunit` in the `require-dev`
section of your `composer.json`.

You can test the PHPUnit installation by running:
```console
./vendor/bin/phpunit --version
```

If for some reason, PHPUnit is not installed refer to the [PHPUnit Installation Guide](https://docs.phpunit.de/en/11.4/installation.html#installing-phpunit-with-composer).

### Writing Functional Tests with PHPUnit

For instructions on generating the factory, please refer to the  [Generate The Factory section](./#generate-the-factory).

#### Writing PHPUnit tests

Here’s an example of a test class, which use the Factory:

```php
<?php

namespace Tests\Feature;

use App\Models\Book;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;
use ApiPlatform\Laravel\Test\ApiTestAssertionsTrait;

class BooksTest extends TestCase
{
    use RefreshDatabase, ApiTestAssertionsTrait;

    /**
     * Test to fetch the collection of books.
     */
    public function testGetCollection(): void
    {
        // Create 100 books using the factory
        Book::factory()->count(100)->create();

        // Send a GET request to the collection endpoint
        $response = $this->getJson('/api/books');

        // Assert that the response is successful (200 OK)
        $response->assertStatus(200);

        // Check the Content-Type header
        $response->assertHeader('Content-Type', 'application/json');

        // Assert the returned JSON contains the expected structure using assertJsonContains from the trait
        $this->assertJsonContains([
            '@context' => '/contexts/Book',
            '@id' => '/books',
            '@type' => 'Collection',
            'totalItems' => 100,
            'view' => [
                '@id' => '/books?page=1',
                '@type' => 'PartialCollectionView',
                'first' => '/books?page=1',
                'last' => '/books?page=4',
                'next' => '/books?page=2',
            ],
        ], $response->json());

        // Assert that 30 items are returned in the response
        $this->assertCount(30, $response->json('data'));
    }

    /**
     * Test to create a valid book.
     */
    public function testCreateBook(): void
    {
        // Send a POST request to create a book
        $response = $this->postJson('/api/books', [
            'isbn' => '0099740915',
            'title' => 'The Handmaid\'s Tale',
            'description' => 'Brilliantly conceived and executed, this powerful evocation of twenty-first century America...',
            'author' => 'Margaret Atwood',
            'publication_date' => '1985-07-31',
        ]);

        // Assert that the book was created successfully (201)
        $response->assertStatus(201);

        // Check the Content-Type header
        $response->assertHeader('Content-Type', 'application/json');

        // Assert the returned JSON contains the expected book information using assertJsonContains
        $this->assertJsonContains([
            '@context' => '/contexts/Book',
            '@type' => 'Book',
            'isbn' => '0099740915',
            'title' => 'The Handmaid\'s Tale',
            'author' => 'Margaret Atwood',
            'publication_date' => '1985-07-31',
            'reviews' => [],
        ], $response->json());

        // Assert that the URI of the created resource matches the expected format
        $this->assertMatchesRegularExpression('~^/api/books/\d+$~', $response->json('@id'));
    }

    /**
     * Test to create an invalid book and validate error handling.
     */
    public function testCreateInvalidBook(): void
    {
        // Send a POST request with invalid data
        $response = $this->postJson('/api/books', [
            'isbn' => 'invalid',
        ]);

        // Assert that the response status is 422 Unprocessable Entity
        $response->assertStatus(422);

        // Check the Content-Type header
        $response->assertHeader('Content-Type', 'application/json');

        // Assert the JSON response contains the validation errors using assertJsonContains
        $this->assertJsonContains([
            '@context' => '/contexts/ConstraintViolationList',
            '@type' => 'ConstraintViolationList',
            'title' => 'An error occurred',
            'description' => [
                'isbn' => 'This value is neither a valid ISBN-10 nor a valid ISBN-13.',
                'title' => 'This value should not be blank.',
                'description' => 'This value should not be blank.',
                'author' => 'This value should not be blank.',
                'publication_date' => 'This value should not be null.',
            ],
        ], $response->json());
    }

    /**
     * Test to update a book.
     */
    public function testUpdateBook(): void
    {
        // Create a book using the factory
        $book = Book::factory()->create(['isbn' => '9781344037075']);

        // Get the IRI of the book using getIriFromResource from the trait
        $iri = $this->getIriFromResource($book);

        // Send a PATCH request to update the book's title
        $response = $this->patchJson($iri, [
            'title' => 'Updated Title',
        ]);

        // Assert that the response is successful (200 OK)
        $response->assertStatus(200);

        // Assert the JSON response contains the updated book information using assertJsonContains
        $this->assertJsonContains([
            '@id' => $iri,
            'isbn' => '9781344037075',
            'title' => 'Updated Title',
        ], $response->json());
    }

    /**
     * Test to delete a book.
     */
    public function testDeleteBook(): void
    {
        // Create a book using the factory
        $book = Book::factory()->create(['isbn' => '9781344037075']);

        // Get the IRI of the book using getIriFromResource from the trait
        $iri = $this->getIriFromResource($book);

        // Send a DELETE request to remove the book
        $response = $this->deleteJson($iri);

        // Assert that the response status is 204 No Content
        $response->assertStatus(204);

        // Assert that the book is no longer in the database
        $this->assertDatabaseMissing('books', ['id' => $book->id]);
    }
}
```

In the example above, the [RefreshDatabase Trait](https://laravel.com/docs/database-testing#resetting-the-database-after-each-test)
is used to ensure that the database is automatically reset between test runs. This guarantees that each test starts with
a clean database state, avoiding conflicts from residual data and ensuring test isolation.

This trait is especially useful when testing operations that modify the database, as it rolls back any changes made
during the test. As a result, your test environment remains reliable and consistent across multiple test executions.

#### Run PHPUnit tests

If everything is working properly, you should see `OK (5 tests, 15 assertions)`.
Your REST API is now properly tested!

Check out the [API Test Assertions section](./#api-test-assertions-with-laravel) to discover the full range of assertions
and other features provided by API Platform's test utilities.

## Writing Unit Tests

In addition to integration tests written using the helpers provided by Pest and PHPUnit, all the classes of your project
should be covered by [unit tests](https://en.wikipedia.org/wiki/Unit_testing).
To do so, learn how to write unit tests with [Pest](https://pestphp.com), [PHPUnit](https://phpunit.de/) and
[Laravel Creating Tests Guide](https://laravel.com/docs/11.x/testing#creating-tests).

## Continuous Integration, Continuous Delivery and Continuous Deployment

Running your test suite in your [CI/CD pipeline](https://en.wikipedia.org/wiki/Continuous_integration) is important to ensure good quality and delivery time.

The API Platform distribution is [shipped with a GitHub Actions workflow](https://github.com/api-platform/api-platform/blob/main/.github/workflows/ci.yml) that builds the Docker images, does a [smoke test](<https://en.wikipedia.org/wiki/Smoke_testing_(software)>) to check that the application's entrypoint is accessible, and runs PHPUnit.

The API Platform Demo [contains a CD workflow](https://github.com/api-platform/demo/tree/main/.github/workflows) that uses [the Helm chart provided with the distribution](../deployment/kubernetes.md) to deploy the app on a Kubernetes cluster.

## Additional and Alternative Testing Tools

You may also be interested in these alternative testing tools (not included in the API Platform distribution):

- [Hoppscotch](https://docs.hoppscotch.io/features/tests), create functional test for your API
- [Behat](https://behat.org), a [behavior-driven development (BDD)](https://en.wikipedia.org/wiki/Behavior-driven_development) framework to write the API specification as user
  stories and in natural language then execute these scenarios against the application to validate its behavior;
- [Playwright](https://playwright.dev) is recommended if you use have PWA/JavaScript-heavy app.

## Testing Utilities for Laravel

### API Test Assertions with Laravel

In addition to [the built-in ones](https://phpunit.readthedocs.io/en/main/assertions.html), API Platform provides
convenient PHPUnit assertions dedicated to API testing:

```php
<?php
// api/tests/MyTest.php

namespace App\Tests;

use ApiPlatform\Laravel\Test\ApiTestAssertionsTrait;
use Tests\TestCase;

class MyTest extends TestCase
{
    use ApiTestAssertionsTrait;
 
    public function testSomething(): void
    {
        // $response = $this->get('/');

        // Asserts that an array has a specified subset.
        $this->assertArraySubset(/* An array or an iterable */);

        // Asserts that the returned JSON is a superset of the passed one
        $this->assertJsonContains(/* a JSON document as an array or as a string */);
    }
}
```

There is also a method to find the IRI matching a given resource:

```php
<?php
// api/tests/BooksTest.php

namespace App\Tests;

use ApiPlatform\Laravel\Test\ApiTestAssertionsTrait;
use App\Models\Book; 
use Tests\TestCase;

class BooksTest extends TestCase
{
    use ApiTestAssertionsTrait;

    public function testFindBook(): void
    {
        $book = Book::firstOrFail();
        $iri = $this->getIriFromResource($book);
        
        $response = $this->get($iri);
        
        $response->assertStatus(200);
    }
}
```
