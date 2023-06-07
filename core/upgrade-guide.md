# Upgrade Guide

## API Platform 2.7/3.0

### I Want to Try the New Metadata System

Note that if you want to use the **new metadata system**, you need to set:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    metadata_backward_compatibility_layer: false
```

This will be the default value in 3.0, in 2.7 it's left to `true` so that nothing breaks by updating.
By doing so you won't get access to legacy services and this will probably break things on code using `api-platform/core:2.6`.

In 3.0, in conformance with the JSON Merge Patch RFC, the default value of the `skip_null_values` property is `true` which means that from now on `null` values are omitted during serialization.
```yaml
api_platform:
  defaults:
    normalization_context:
      skip_null_values: true
```

### I'm Migrating From 2.6 and Want to Prepare For 3.0

1. Update the code to 2.7: `composer require api-platform/core:^2.7`
2. Take care of the deprecations and update your code to the new interfaces, documented on this page
3. Switch the `metadata_backward_compatibility_layer` flag to `false`
4. Use the [`api:upgrade-resource` command](#the-upgrade-command)

Read more about the `metadata_backward_compatibility_layer` flag [here](#the-metadata_backward_compatibility_layer-flag).

## Changes

### Summary of the Changes Between 2.6 And 2.7/3.0

- New Resource metadata allowing to declare multiple Resources on a class: `ApiPlatform\Metadata\ApiResource`
- Clarification of some properties within the ApiResource declaration
- Removal of item and collection differences on operation declaration
- `ApiPlatform\Core\DataProvider\...DataProviderInterface` has a new
interface `ApiPlatform\State\ProviderInterface`
- `ApiPlatform\Core\DataPersister\...DataPersisterInterface` has a new
interface `ApiPlatform\State\ProcessorInterface`
- New ApiProperty metadata `ApiPlatform\Metadata\ApiProperty`
- Configuration flag `metadata_backward_compatibility_layer` that allows the use of legacy metadata layers
- `ApiPlatform\Core\DataTransformer\DataTransformerInterface` is deprecated and will be removed in 3.0
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
    iri: 'https://schema.org/Book',
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

#[ApiResource(types: ['https://schema.org/Book'], operations: [
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

### Removal of Item/Collection Operations

We removed the notion of item and collection.
Instead, use HTTP verbs matching the operation you want to declare.
There is also a `collection` flag instructing whether the
operation returns an array or an object.
The default `ApiResource` attribute still declares a CRUD:

```php
#[ApiResource]
```

is the same as:

```php
#[ApiResource(operations: [
    new Get(),
    new Put(),
    new Patch(),
    new Delete(),
    new GetCollection(),
    new Post(),
])]
```

### Metadata Changes

#### #[ApiResource]

`ApiPlatform\Metadata\ApiResource` instead of `ApiPlatform\Core\Annotation\ApiResource`

|Before|After|
|---|---|
|`iri: 'https://schema.org/Book'`|`types: ['https://schema.org/Book']`|
|`path: '/books/{id}/publication'`|`uriTemplate: '/books/{id}/publication'`|
|`identifiers: []`|`uriVariables: []`|
|`attributes: []`|`extraProperties: []`|
|`attributes: ['validation_groups' => ['a', 'b']]`|`validationContext: ['groups' => ['a', 'b']]`|

#### #[ApiProperty]

`ApiPlatform\Metadata\ApiProperty` instead of `ApiPlatform\Core\Annotation\ApiProperty`

|Before|After|
|---|---|
|`iri: 'https://schema.org/Book'`|`types: ['https://schema.org/Book']`|
|`type: 'string'`|`builtinTypes: ['string']`|

Note that `builtinTypes` are computed automatically from PHP types.

For example:

```php
class Book
{
    public string|Isbn $isbn;
}
```

Will compute: `builtinTypes: ['string', Isbn::class]`

### The `metadata_backward_compatibility_layer` Flag

In 2.7 the `metadata_backward_compatibility_layer` flag is set to `true`.
This means that all the legacy services will still work just as they used
to work in 2.6 (for example `PropertyMetadataFactoryInterface` or
`ResourceMetadataFactoryInterface`).
When updating we advise to first resolve the deprecations then to set this
flag to `false` to use the new metadata system.

When `metadata_backward_compatibility_layer` is set to `false`:
- there's still a bridge with the legacy `ApiPlatform\Core\Annotation\ApiResource` and old metadata will still work
- the deprecated Symfony services will have their interface changed (for example `ApiPlatform\Core\Api\IriConverterInterface`
will be `ApiPlatform\Api\IriConverterInterface`) and it may break your dependency injection.
- the new metadata system is available `ApiPlatform\Metadata\ApiResource`

### SearchFilter

If you want to use the new namespaces for the search filter
(`ApiPlatform\Doctrine\Orm\Filter\SearchFilter` instead of`ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter` or
`ApiPlatform\Doctrine\Odm\Filter\SearchFilter` instead of`ApiPlatform\Core\Bridge\Doctrine\Odm\Filter\SearchFilter`) you
need to set the `metadata_backward_compatibility_layer` to `false` as this filter relies on the implementation
of the new `ApiPlatform\Api\IriConverterInterface`.

In 3.0 this flag will default to `false` and the legacy code will be removed.

## The Upgrade Command

The upgrade command will automatically upgrade the old `ApiPlatform\Core\Annotation\ApiResource` to `ApiPlatform\Metadata\ApiResource`.
By default, this does a dry run and shows a diff:

```bash
php bin/console api:upgrade-resource
```

To write in-place use the `force` option:

```bash
php bin/console api:upgrade-resource -f
```

## Providers/Processors

Providers and Processors are replacing DataProviders and DataPersisters.
We reduced their interface to only one method and the class used by your operation can be specified directly within the metadata.
Using Doctrine, a default resource would use these:

```php

<?php

use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\Delete;
use ApiPlatform\Metadata\Post;
use ApiPlatform\Metadata\Put;

#[Put(processor: ApiPlatform\Doctrine\Common\State\PersistProcessor::class, provider: ApiPlatform\Doctrine\Orm\State\ItemProvider::class)]
#[Post(processor: ApiPlatform\Doctrine\Common\State\PersistProcessor::class)]
#[Delete(processor: ApiPlatform\Doctrine\Common\State\RemoveProcessor::class)]
#[Get(provider: ApiPlatform\Doctrine\Orm\State\ItemProvider::class)]
#[GetCollection(provider: ApiPlatform\Doctrine\Orm\State\CollectionProvider::class)]
class Book {}
```

See also the respective documentation:

- [State Processor](./state-processors.md)
- [State Provider](./state-providers.md)

## DataTransformers and DTO Support

Data transformers have been deprecated, instead you can still document the `output` or the `input` DTO.
Then, just handle the `input` in a custom [State Processor](./state-processors.md) or return another `output` in a custom [State Provider](./state-providers.md).

The [dto documentation](./dto.md) has been adapted accordingly.
