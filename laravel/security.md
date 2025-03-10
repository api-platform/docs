# Security with Laravel

## Policies

API Platform is compatible with Laravel's [authorization](https://laravel.com/docs/authorization) mechanism.

To utilize policies in API Platform, it is essential to have Laravel's authentication system initialized.
See the [Authentication section](./#authentication) for more information.

Once a gate is defined, API Platform will automatically detect your policy.

```php
// app/Models/Book.php

use ApiPlatform\Metadata\Patch;

#[Patch]
class Book extends Model
{
}
```

API Platform will detect the operation and map it to a specific method in your policy according to the rules defined in
this table:

| Operation      | Policy                                                     |
| -------------- | ---------------------------------------------------------- |
| GET collection | `viewAny`                                                  |
| GET            | `view`                                                     |
| POST           | `create`                                                   |
| PATCH          | `update`                                                   |
| DELETE         | `delete`                                                   |
| PUT            | `update` or `create` if the resource doesn't already exist |

If your policy methods do not match Laravel's conventions, you can always use the `policy` property on an operation
attribute to enforce this policy:

```php
// app/Models/Book.php
namespace App\Models;

 use ApiPlatform\Metadata\ApiResource;
+use ApiPlatform\Metadata\Patch;
 use Illuminate\Database\Eloquent\Model;

-#[ApiResource]
 #[ApiResource(
     paginationItemsPerPage: 10,
+    operations: [
+       new Patch(
+            policy: 'myCustomPolicy',
+       ),
+    ],
)]
 class Book extends Model
 {
 }
```

You also can link a model to a policy:

```php
use App\Models\Book;
use App\Tests\Book\BookPolicy;
use Illuminate\Support\Facades\Gate;

Gate::guessPolicyNamesUsing(function (string $modelClass): ?string {
    return Book::class === $modelClass ?
        BookPolicy::class :
        null;
});
```

## Authentication

Usually, you will use [Sanctum](https://laravel.com/docs/sanctum) and add a middleware on secured routes:

```php
// app/Models/Book.php

use ApiPlatform\Metadata\Patch;

#[Patch(middleware: 'auth:sanctum')]
class Book extends Model
{
}
```

Or you can define it globally in the configuration by adding the following code:

```php
<?php
// config/api-platform.php
return [
    // ....
    'defaults' => [
        // ....
        'middleware' => 'auth:sanctum',
    ],
];
```
