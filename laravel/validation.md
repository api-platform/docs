# Validation

You can add [validation rules](https://laravel.com/docs/validation) within the `rules` option:

```php
// app/Models/Book.php

use ApiPlatform\Metadata\ApiResource;

#[ApiResource(
    rules: [
        'title' => 'required',
    ]
)]
class Book extends Model
{
}
```
