# Upgrade Guide

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
