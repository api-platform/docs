# Security

API platform is compatible with Laravel [authorization](https://laravel.com/docs/authorization) mechanism. Once a gate is defined, you can specify the policy to use within an operation:

```php
// app/Models/Book.php 

use ApiPlatform\Metadata\Patch;

#[Patch(policy: 'update')]
class Book extends Model
{
}
```

Usually, you will use [Sanctum](https://laravel.com/docs/sanctum) and add a middleware on secured routes:

```php
// app/Models/Book.php 

use ApiPlatform\Metadata\Patch;

#[Patch(middleware: 'auth:sanctum', policy: 'update')]
class Book extends Model
{
}
```
