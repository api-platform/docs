# JWT Authentication with Laravel

> [!NOTE]
> While solutions like `tymondesigns/jwt-auth` (Laravel) or `LexikJWTAuthenticationBundle` (Symfony) are popular,
> **we recommend adopting open standards such as [OpenID Connect (OIDC)](https://openid.net/connect/)** for robust, scalable,
> and interoperable authentication.

For comprehensive details on authentication, refer to our [Laravel Authentication documentation](../laravel/index.md#authentication).

## Setup Instructions

1. **Install**  
   Follow the official installation guide of [Laravel Passport](https://laravel.com/docs/passport#installation) to implement
   OpenID Connect (OIDC) standards in your Laravel application.
   Alternatively, if you prefer an ad-hoc solution, you can use [tymondesigns/jwt-auth](https://github.com/tymondesigns/jwt-auth)
   to set up JWT authentication in your Laravel project.

2. **Configure Authentication**  
   Refer to the [Authentication section](../laravel/index.md#authentication) of our documentation to properly configure
   and secure your API with JWT tokens.

> [!TIP]
> Use [Laravel middlewares with API Platform](../laravel/index.md#middlewares) such as `auth:api` to
> restrict access to certain endpoints, ensuring only authenticated users can access them.

By following these steps, you can set up a secure and scalable JWT-based authentication system in your Laravel application.

## Testing

To verify your authentication setup using `ApiTestCase`, you can write a test method tailored to your preferred testing
framework. Here's how you can approach it for both **Pest** and **PHPUnit**:

> [!NOTE]
> Ensure your routes (/api/auth) and authentication mechanisms are configured to match your application's implementation.

### Test with Pest

```php
<?php

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

it('authenticates a user and tests protected endpoints', function () {
    // Create a user
    $user = User::factory()->create([
        'email' => 'test@example.com',
        'password' => bcrypt('$3CR3T'), // Hash the password
    ]);

    // Retrieve a token
    $response = $this->postJson('/api/auth', [
        'email' => 'test@example.com',
        'password' => '$3CR3T',
    ]);

    $response->assertStatus(200)
        ->assertJsonStructure(['token']);

    $token = $response->json('token');

    // Test not authorized
    $this->getJson('/api/greetings')
        ->assertStatus(401);

    // Test authorized
    $this->withHeader('Authorization', "Bearer $token")
        ->getJson('/api/greetings')
        ->assertStatus(200);
});
```

### Test with PHPUnit

```php
<?php

namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class AuthenticationTest extends TestCase
{
    use RefreshDatabase;

    public function testLogin(): void
    {
        $user = User::factory()->create([
            'email' => 'test@example.com',
            'password' => bcrypt('$3CR3T'), // Hash the password
        ]);

        // Retrieve a token
        $response = $this->postJson('/api/auth', [
            'email' => 'test@example.com',
            'password' => '$3CR3T',
        ]);

        $response->assertStatus(200)
            ->assertJsonStructure(['token']);

        $token = $response->json('token');

        // Test not authorized
        $this->getJson('/api/greetings')
            ->assertStatus(401);

        // Test authorized
        $this->withHeader('Authorization', "Bearer $token")
            ->getJson('/api/greetings')
            ->assertStatus(200);
    }
}
```


