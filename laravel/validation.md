# Validation with Laravel

API Platform simplifies the validation of data sent by clients to the API, typically user inputs
submitted through forms.

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

## Constraint-Aware 422 for Denormalization Errors

Starting with API Platform 4.4, type mismatches detected during input denormalization (for example,
the client sends `"foo"` for an `int` field, or `null` for a non-nullable property) are promoted to
HTTP 422 validation responses when the affected property has a matching Laravel validation rule.
When no matching rule exists, API Platform rethrows the original serializer exception as an honest
HTTP 400.

This eliminates the need to write a custom middleware or exception handler solely to convert 400
serializer errors into 422 validation responses.

### How It Works

`DeserializeProvider` catches denormalization exceptions from the Symfony Serializer. It delegates
to `ApiPlatform\Laravel\State\DenormalizationViolationFactory`, which reads the `rules` declared on
the operation and applies the following rule table:

| Serializer `currentType` | Matching rule in `rules` | Code |
| --- | --- | --- |
| `null` | `required`, `filled` | `blank` |
| `null` | `present` | `null` |
| any wrong type | `string`, `integer`, `int`, `numeric`, `boolean`, `bool`, `array`, `date`, `json` | `invalid_type` |
| any wrong type | any other rule (when `nullable` is absent) | `invalid_type` |
| `null` | `nullable` only (no `required`, `present`, or `filled`) | 400 (rethrow) |
| any | (no rule for the property) | 400 (rethrow) |

Rules may be declared in string pipe-separated form (`'required|integer'`) or array form
(`['required', 'integer']`). Object-based rules (`Rule`, `ValidationRule`) and
`FormRequest`-class rule sets are skipped — `FormRequest` contracts run during the
validation phase against the raw request, not the denormalized body.

### Example

```php
// app/Models/Book.php

use ApiPlatform\Metadata\ApiResource;
use Illuminate\Database\Eloquent\Model;

#[ApiResource(
    rules: [
        'title' => 'required|string',
        'year'  => 'required|integer',
    ]
)]
class Book extends Model
{
    protected $fillable = ['title', 'year'];
}
```

Sending `null` for the `year` field:

```http
POST /api/books HTTP/1.1
Content-Type: application/json

{"title": "Dune", "year": null}
```

Returns 422 with `blank` code because `required` is present:

```json
{
    "type": "/validation_errors/abc123",
    "title": "Validation Error",
    "description": "year: This value should not be blank.",
    "status": 422,
    "violations": [
        {
            "propertyPath": "year",
            "message": "This value should not be blank.",
            "code": "blank"
        }
    ]
}
```

Sending a string for the `year` field:

```http
POST /api/books HTTP/1.1
Content-Type: application/json

{"title": "Dune", "year": "nineteen-sixty-five"}
```

Returns 422 with `invalid_type` code because `integer` is present:

```json
{
    "type": "/validation_errors/def456",
    "title": "Validation Error",
    "description": "year: This value should be of type integer.",
    "status": 422,
    "violations": [
        {
            "propertyPath": "year",
            "message": "This value should be of type integer.",
            "code": "invalid_type"
        }
    ]
}
```

If the `year` property had no rule at all, both requests would receive HTTP 400 instead.

### Nullable Fields

A field declared as `nullable` without `required`, `present`, or `filled` explicitly permits `null`
values, so a `null` submission for such a field is not promoted to 422 and rethrows the original
400:

```php
#[ApiResource(
    rules: [
        'publishedAt' => 'nullable|date',
    ]
)]
```

Sending `null` for `publishedAt` with only `nullable|date` produces HTTP 400, not 422.

### Relationship with Symfony Validation

The constraint-aware 422 behavior described above operates on the Laravel rules defined on the
operation. It is independent from the Symfony Validator stack. For the equivalent Symfony
integration, see the
[Validation with Symfony documentation](../symfony/validation.md#constraint-aware-422-for-denormalization-errors).
