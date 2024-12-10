# Validation with Laravel

API Platform simplifies the validation of data sent by clients to the API, typically user inputs submitted through forms.

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
