# Bootstrapping the Core Library

You may want to run a minimal version of API Platform. This one file runs API Platform (without
GraphQL, Eloquent, Doctrine MongoDB...). It requires the following Composer packages:

> [!NOTE]
> This documentation is a work in progress we're working on improving it, in the mean time
> we declare most of the services manually in the
> [ApiPlatformProvider](https://github.com/api-platform/core/blob/64768a6a5b480e1b8e33c639fb28b27883c69b79/src/Laravel/ApiPlatformProvider.php)
> it can be source of inspiration.

## Components

API Platform is installable as a set of components, for example:

```console
composer require \
    api-platform/serializer \
    api-platform/metadata \
    api-platform/state \
    api-platform/jsonld \
    phpdocumentor/reflection-docblock \
    symfony/property-info \
    symfony/routing \
    symfony/validator
```
