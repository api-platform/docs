# Upgrade Guide

## What Has Changed Between 2.6 And 2.7?

- New Resource metadata allowing to declare multiple Resources on a class: `ApiPlatform\Metadata\ApiResource`
- Clarification of some properties within the ApiResource declaration
- Removal of the item and collection difference on operation declaration
- `ApiPlatform\Core\DataProvider\...DataProviderInterface` has a new
interface `ApiPlatform\State\ProviderInterface`
- `ApiPlatform\Core\DataPersister\...DataPersisterInterface` has a new
interface `ApiPlatform\State\ProcessorInterface`
- New ApiProperty metadata `ApiPlatform\Metadata\ApiProperty`
- Configuration flag `metadata_backward_compatibility_layer` that allows
the use of legacy metadata layers
- Subresources are now additional resources marked with an `#[ApiResource]` attribute (see [the new subresource documentation](./subresources.md))

The detailed changes are present in the [CHANGELOG](https://github.com/api-platform/core/blob/main/CHANGELOG.md).

### ApiResource Metadata

The `ApiResource` annotation has a new namespace:
`ApiPlatform\Metadata\ApiResource` instead of `ApiPlatform\Core\Annotation\ApiResource`.

For example, the Book resource in 2.6:

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(
    iri: 'http://schema.org/Book',
    itemOperations: [
        'get',
        'post_publication' => [
            'method' => 'POST',
            'path' => '/books/{id}/publication',
        ],
    ])
]
class Book
{
    // ...
}
```

Becomes in 2.7:

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\Post;
use App\Controller\CreateBookPublication;

#[ApiResource(types: ['http://schema.org/Book'], operations: [
    new Get(),
    new Post(name: 'publication', uriTemplate: '/books/{id}/publication')
])]
class Book
{
    // ...
}
```

You can use the `api:upgrade-resource` command to upgrade
your resources automatically, [see instructions here](#the-upgrade-command).

### Removal of Item/Collection operations

We removed the notion of item and collection. Instead, use
http verbs matching the operation you want to declare.
There is also a `collection` flag instructing wether the
operation returns an array or an object.
The default ApiResource attribute still declares a CRUD:

```php
#[ApiResource]
```

is the same as

```php
#[ApiResource(operations=[
    new Get(),
    new Put(),
    new Patch(),
    new Delete(),
    new GetCollection(),
    new Post()
])]
```

### Detailed metadata changes

#### #[ApiResource]

`ApiPlatform\Metadata\ApiResource` instead of `ApiPlatform\Core\Annotation\ApiResource`

|Before|After|
|---|---|
|`iri: 'http://schema.org/Book'`|`types: ['http://schema.org/Book']`|
|`path: '/books/{id}/publication'`|`uriTemplate: '/books/{id}/publication'`|
|`identifiers: []`|`uriVariables: []`|
|`attributes: []`|`extraProperties: []`|
|`attributes: ['validation_groups' => ['a', 'b']]`|`validationContext: ['groups' => ['a', 'b']]`|

#### #[ApiProperty]

`ApiPlatform\Metadata\ApiProperty` instead of `ApiPlatform\Core\Annotation\ApiProperty`

|Before|After|
|---|---|
|`iri: 'http://schema.org/Book'`|`types: ['http://schema.org/Book']`|
|`type: 'string'`|`builtinTypes: ['string']`|

Note that builtinTypes are computed automatically from php types.

For example:

```php
class Book
{
    public string|Isbn $isbn;
}
```

Will compute: `builtinTypes: ['string', Isbn::class]`

## Versions 2.7 and 3.0

In 2.7 the `metadata_backward_compatibility_layer` flag is set to `true`.
This means that all the legacy services will still work just as they used
to work in 2.6 (for example `PropertyMetadataFactoryInterface` or
`ResourceMetadataFactoryInterface`). When updating we advise to first
resolve the deprecations then to set this flag to `false` to use the
new metadata system.

### SearchFilter

If you want to use the new namespaces for the search filter
(`ApiPlatform\Doctrine\Orm\Filter\SearchFilter` instead of`ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter` or
`ApiPlatform\Doctrine\Odm\Filter\SearchFilter` instead of`ApiPlatform\Core\Bridge\Doctrine\Odm\Filter\SearchFilter`) you
need to use the `metadata_backward_compatibility_layer` to false as this filter relies on the implementation
of the new `ApiPlatform\Api\IriConverterInterface`.

In 3.0 this flag will default to `false` and the legacy code will be removed.

## The upgrade command

The upgrade command will automatically upgrade the old `ApiPlatform\Core\Annotation\ApiResource` to `ApiPlatform\Metadata\ApiResource`.
By default this does a dry run and shows a diff:

```bash
php bin/console api:upgrade-resource
```

To write in-place use the `force` option:

```bash
php bin/console api:upgrade-resource -f
```
