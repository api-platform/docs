# Upgrade Guide

## API Platform 3.4

Remove the `keep_legacy_inflector`, the `event_listeners_backward_compatibility_layer` and the `rfc_7807_compliant_errors` flag:

```diff
api_platform:
-        event_listeners_backward_compatibility_layer: false
-        keep_legacy_inflector: false
        extra_properties:
-            standard_put: true
-            rfc_7807_compliant_errors: true
```

If you use a custom normalizer for validation exception use:

```yaml
api_platform:
  validator:
    legacy_validation_exception: true
```

Indeed, we will throw another validation class in API Platform 4 we will throw `ApiPlatform\Validator\Exception\ValidationException` instead of `ApiPlatform\Symfony\Validator\Exception\ValidationException`

It's really important to add the `use_symfony_listeners` flag, set to `true` if you use Symfony listeners or controllers:

```yaml
api_platform:
  use_symfony_listeners: false
```

The `keep_legacy_inflector` flag will be removed from API Platform 4, you need to fix your issues first. In API Platform 3.4, the Inflector is available as a service that you can configure through:

```yaml
api_platform:
  inflector: api_platform.metadata.inflector
```

Implement the `ApiPlatform\Metadata\InflectorInterface` if you need to tweak its behavior.

We added an `hydra_prefix` configuration as the `hydra:` prefix will be removed by default in API Platform 4:

```yaml
api_platform:
  serializer:
    hydra_prefix: false
```

Standard PUT is now `true` by default, you can change its value using:

```yaml
api_platform:
  extra_properties:
    standard_put: true
```

We recommend using the standalone API Platform packages instead of the Core monolithic repository.

Update your `composer.json` like that:

```patch
 {
     "require": {
-        "api-platform/core": "^3",
+        "api-platform/symfony": "^3 || ^4"
+        // also add the extra packages you need, like "api-platform/doctrine-orm"
     }
 }
```

## API Platform 3.1/3.2

This is the recommended configuration for API Platform 3.2. We review each of these changes in this document.

```yaml
api_platform:
  title: Hello API Platform
  version: 1.0.0
  formats:
    jsonld: ['application/ld+json']
  docs_formats:
    jsonld: ['application/ld+json']
    jsonopenapi: ['application/vnd.openapi+json']
    html: ['text/html']
  defaults:
    stateless: true
    cache_headers:
      vary: ['Content-Type', 'Authorization', 'Origin']
    extra_properties:
      standard_put: true
      rfc_7807_compliant_errors: true
  event_listeners_backward_compatibility_layer: false
  keep_legacy_inflector: false
```

### Formats

We noticed that API Platform was enabling `json` by default because of our OpenAPI support. We introduced the new `application/vnd.openapi+json`. Therefore if you want `json` you need to explicitly handle it:

```yaml
formats:
  json: ['application/json']
```

You can also remove documentations you're not using via the new `docs_formats`.

A new option `error_formats` is also used for content negotiation.

### Event listeners

For new users we recommend to use

```yaml
event_listeners_backward_compatibility_layer: false
```

This allows API Platform to not use http kernel event listeners. It also allows you to force options like `read: true` or `validate: true`. This simplifies use cases like [validating a delete operation](/docs/v3.2/guides/delete-operation-with-validation/)
Event listeners will not get removed and are not deprecated, they'll use our providers and processors in a future version.

### Inflector

We're switching to `symfony/string` [inflector](https://symfony.com/doc/current/components/string.html#inflector), to keep using `doctrine/inflector` use:

```yaml
keep_legacy_inflector: true
```

We strongly recommend that you use your own inflector anyways with a [PathSegmentNameGenerator](https://github.com/api-platform/core/blob/f776f11fd23e5397a65c1355a9ebcbb20afac9c2/src/Metadata/Operation/UnderscorePathSegmentNameGenerator.php).

### Errors

```yaml
defaults:
  extra_properties:
    rfc_7807_compliant_errors: true
```

As this is an `extraProperties` it's configurable per resource/operation. This is improving the compatibility of Hydra errors with JSON problem. It also enables new extension points on [Errors](https://api-platform.com/docs/v3.2/core/errors/) such as [Error provider](https://api-platform.com/docs/v3.2/guides/error-provider/) and [Error Resource](https://api-platform.com/docs/v3.2/guides/error-resource/).
