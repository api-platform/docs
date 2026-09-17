# Upgrade Guide

## API Platform 4.4 to 5.0

5.0 removes long-deprecated APIs. Components ship with a `@beta` stability flag (for example
`"api-platform/state": "^5.0@beta"`) instead of `@alpha`. If your 4.4 install runs without
deprecation notices, most of this upgrade is a no-op; the sections below cover the changes that are
not announced by a 4.4 deprecation.

### API Platform 5.0 Breaking Changes

#### JSON:API `use_iri_as_id` Now Defaults to `false`

The default announced by the 4.4 deprecation now applies: not setting
`api_platform.jsonapi.use_iri_as_id` explicitly resolves to `false` instead of `true`. The JSON:API
`data.id` member carries the resource identifier and the IRI moves to `data.links.self`.

**Before (4.x default, `use_iri_as_id: true`)**:

```json
{
    "data": {
        "id": "/dummies/10",
        "type": "Dummy",
        "attributes": {
            "name": "Dummy #10"
        }
    }
}
```

**After (5.0 default, `use_iri_as_id: false`)**:

```json
{
    "data": {
        "id": "10",
        "type": "Dummy",
        "links": {
            "self": "/dummies/10"
        },
        "attributes": {
            "name": "Dummy #10"
        }
    }
}
```

To keep the previous payload, set the option explicitly back to `true`.

**Symfony**:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    jsonapi:
        use_iri_as_id: true
```

**Laravel**:

```php
// config/api-platform.php
return [
    'jsonapi' => [
        'use_iri_as_id' => true,
    ],
];
```

See [JSON:API](jsonapi.md#entity-identifiers-as-resource-ids) for the full behavior, including
composite identifiers and resources without a standalone item endpoint.

#### `DeserializeProvider` No Longer Accepts a Translator

`ApiPlatform\State\Provider\DeserializeProvider` drops the
`Symfony\Contracts\Translation\TranslatorInterface` fourth constructor argument that was deprecated
in 4.4. `DenormalizationViolationFactoryInterface`, previously the fifth argument, moves to the
fourth position:

```php
public function __construct(
    ?ProviderInterface $decorated,
    SerializerInterface $serializer,
    SerializerContextBuilderInterface $serializerContextBuilder,
    ?DenormalizationViolationFactoryInterface $violationFactory = null,
)
```

**Who is affected**: anyone constructing `DeserializeProvider` by hand, or overriding the
`api_platform.state_provider.deserialize` service definition with a `TranslatorInterface` argument.
Translation of denormalization violations is entirely handled by
`DenormalizationViolationFactoryInterface` since 4.4; drop the translator argument and shift any
positional `$violationFactory` argument one position to the left.

`api-platform/state` no longer requires `symfony/translation-contracts`.

#### Deprecated APIs Removed

The following long-deprecated APIs are removed:

- **Configuration keys** — these Symfony bundle options no longer exist; remove them from
  `config/packages/api_platform.yaml`. The [configuration reference](configuration.md) lists the
  current options:

    | Removed key                            | Replacement                                                                                                         |
    | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
    | `validator.query_parameter_validation` | none — always on                                                                                                    |
    | `enable_link_security`                 | none — sub-resource link security is always enabled                                                                 |
    | `resource_class_directories`           | none — `#[ApiResource]` classes are autoconfigured                                                                  |
    | `graphql.graphql_playground`           | none — was already ignored                                                                                          |
    | `http_cache.invalidation.varnish_urls` | `http_cache.invalidation.urls` or `scoped_clients`                                                                  |
    | `http_cache.invalidation.xkey`         | a custom purger, see [HTTP cache invalidation](performance.md#enabling-the-built-in-http-cache-invalidation-system) |

- **`ObjectMapperProcessor`** — `ApiPlatform\State\Processor\ObjectMapperProcessor` is removed. Use
  `ApiPlatform\State\Processor\ObjectMapperInputProcessor` and
  `ApiPlatform\State\Processor\ObjectMapperOutputProcessor` instead; see [DTOs](dto.md) for the
  split responsibilities.

- **The `$distinctFormats` constructor argument** — it is removed from both
  `ApiPlatform\JsonSchema\DefinitionNameFactory` and `ApiPlatform\JsonSchema\SchemaFactory`. JSON
  Schema definition names for formats other than `json` and `merge-patch+json` are now always
  suffixed with the format. If you instantiate `SchemaFactory` positionally, its
  `$definitionNameFactory` argument moves from the seventh to the sixth position.

- **`ValidationException`'s string-message constructor** — the first constructor argument only
  accepts a `Symfony\Component\Validator\ConstraintViolationListInterface` now; the
  `string|ConstraintViolationListInterface` union and the plain-string code path are removed.

- **`ApiTestCase::$alwaysBootKernel` defaulting to `null`** — the property now defaults to `false`
  (kernel not rebooted between requests if already booted) instead of triggering a deprecation and
  always booting the kernel. Set it to `true` in your test class if you relied on the implicit
  always-boot behavior.

- **Automatic short-name deduplication** — the
  `defaults.extra_properties.deduplicate_resource_short_names` opt-in flag is removed. Resources
  sharing a `shortName` are now always deduplicated with a numeric suffix (`AttributeResource2`,
  `Employee3`, …), unconditionally, instead of raising a deprecation when two `#[ApiResource]`
  attributes shared the same `shortName` without opting in.

- **Explicit `api_assign_object_to_populate` context** — `DeserializeProvider` no longer falls back
  to computing `SerializerContextBuilderInterface::ASSIGN_OBJECT_TO_POPULATE`
  (`api_assign_object_to_populate`) from the HTTP method itself; it only assigns the loaded object
  to the denormalization context's `object_to_populate` when that flag is already set. Symfony's
  `MainController` and `DeserializeListener`, and Laravel's `ApiPlatformController`, already set it
  for `POST`, `PATCH`, and non-standard `PUT` before calling the provider, so this only matters if
  you call `DeserializeProvider::provide()` directly from a custom controller or pipeline.

- **Serializer-aware state providers** — `ApiPlatform\State\SerializerAwareProviderInterface` and
  `ApiPlatform\State\SerializerAwareProviderTrait` are removed. Inject
  `Symfony\Component\Serializer\SerializerInterface` through your provider's constructor instead of
  relying on `setSerializerLocator()`. The internal `DataProviderPass` that performed setter
  injection for these providers is removed as a consequence.

See [core#8367](https://github.com/api-platform/core/pull/8367) for the full diff.

#### Legacy PropertyInfo Type System Removed

The legacy `symfony/property-info` `Type` system is replaced by `symfony/type-info` throughout:

- `ApiProperty::$builtinTypes`, `ApiProperty::getBuiltinTypes()`, and
  `ApiProperty::withBuiltinTypes()` are removed. Use `ApiProperty::getNativeType()` /
  `withNativeType()`, which return a `Symfony\Component\TypeInfo\Type`.
- GraphQL's `TypeConverterInterface::convertType()` is removed. Implement `convertPhpType()`
  instead; it takes a `Symfony\Component\TypeInfo\Type` rather than a legacy
  `Symfony\Component\PropertyInfo\Type`.
- `ContextAwareTypeBuilderInterface::isCollection(LegacyType $type)` is removed entirely; collection
  detection is handled internally from the `Type` object.

**Who is affected**: custom `TypeConverterInterface` or `ContextAwareTypeBuilderInterface`
implementations, and any code reading `ApiProperty::getBuiltinTypes()`.

#### `PropertyAwareFilterInterface::getProperties()` Is Now a Real Interface Method

`getProperties(): ?array` was previously only documented via an `@method` docblock annotation on
`ApiPlatform\Doctrine\Common\Filter\PropertyAwareFilterInterface` (with the real method commented
out). It is now declared on the interface. Custom filters implementing
`PropertyAwareFilterInterface` without a `getProperties()` method now fail with a fatal "must
implement" error; add the method (or use `PropertyAwareFilterTrait`).

#### JSON:API Error `status` Is Now a String

`ApiPlatform\JsonApi\Serializer\ErrorNormalizer` now always casts the `status` member of a JSON:API
error object to a string, matching the
[JSON:API error object spec](https://jsonapi.org/format/#error-objects). Update any client or custom
normalizer that expects `status` to be an integer.

#### JSON-LD `/contexts/Error` and `/contexts/ConstraintViolationList` Are No Longer Special-Cased

`ApiPlatform\JsonLd\Action\ContextAction::RESERVED_SHORT_NAMES` and the hardcoded base-context
fallback for the `Error` and `ConstraintViolationList` short names are removed. Since exceptions and
validation errors have been resources since 3.2, their `@context` is now built through the normal
per-resource context loop like any other resource. The `/contexts/ConstraintViolationList` route no
longer has a producer: responses reference `/contexts/ConstraintViolation` (singular) instead.

**Who is affected**: code or tests hardcoding `/contexts/ConstraintViolationList` URLs.

#### `SerializerContextBuilder` No Longer Injects `uri_variables`

`SerializerContextBuilder::createFromRequest()` no longer populates `$context['uri_variables']` from
the request attributes. The key is still set — but only later in the state pipeline, by
`SerializeProcessor` and `DeserializeProvider`, which already had the correctly parsed values.

**Who is affected**: custom normalizers reading `$context['uri_variables']` in a context built
directly from `SerializerContextBuilder::createFromRequest()` outside the standard state pipeline
(for example, a custom controller that calls it directly). Normalizers invoked through the regular
provider/processor flow are unaffected — the key is still present by the time normalization runs.

#### `UniqueConstraintViolationException` Maps to 422 by Default

`Doctrine\DBAL\Exception\UniqueConstraintViolationException` is added to the Symfony bundle's
default `exception_to_status` map, resolving to `422 Unprocessable Entity` (alongside the
pre-existing `OptimisticLockException => 409 Conflict`). If you previously mapped this exception
yourself, or relied on it falling through to `500`, review your `exception_to_status` configuration.
See [Exception to status](errors.md#exception-to-status).

#### Doctrine Filters: `RangeFilter` Deprecated, `ComparisonFilter` Gains `[between]`

`ComparisonFilter` now natively supports `?price[between]=10..100`, covering the full range syntax
(`[gt]`/`[gte]`/`[lt]`/`[lte]`/`[between]`). `RangeFilter` is deprecated in favor of it and is
removed in 6.0; `DateFilter` and `ExistsFilter` become standalone classes (they no longer extend
`AbstractFilter`) with no change to their URL syntax. See
[Doctrine Filters](doctrine-filters.md#comparison-filter) for the full migration path.

### API Platform 5.0 Deprecations

#### `FilterInterface::getDescription()` Stays Deprecated

`ApiPlatform\Metadata\FilterInterface::getDescription()` was deprecated in 4.2 for removal in 6.0.
That removal is **deferred to 6.0** and does **not** happen in 5.0 — the method, `#[ApiFilter]`,
`Operation::$filters`, and the `AbstractFilter` base class all still work in 5.0. Only migrate off
them when you are ready to keep pace with the 6.0 timeline; see
[Doctrine Filters](doctrine-filters.md#creating-custom-doctrine-orm-filters) for the modern filter
interfaces.

### API Platform 5.0 Package Changes

The testing utilities have moved from `api-platform/symfony` to the new `api-platform/test` package.
Install it as a development dependency:

```console
composer require --dev api-platform/test:^5.0@beta
```

Update imports to use the new namespace:

```php
use ApiPlatform\Test\ApiTestCase;
```

`ApiPlatform\Symfony\Bundle\Test\ApiTestCase` remains as a deprecated compatibility shim when
`api-platform/test` is installed, but it emits a deprecation notice. The other testing utilities
have moved under the same `ApiPlatform\Test` namespace.

The removal of the internal `Request::getContentType()` fallbacks and the Symfony 6 value-resolver
compatibility interface requires no application changes.

## API Platform 4.3 to 4.4

4.4 is the last 4.x minor. It ships a single backwards-incompatible change (below); everything else
is a deprecation that keeps working until it is removed in a later major (5.0 or 6.0). Fixing the
deprecations now makes the upgrade to the next major a no-op.

### Backwards-Incompatible Changes

#### Denormalization Type Errors on Unconstrained BackedEnum Properties Revert to HTTP 400

Prior to 4.4, `BackedEnum`-typed properties received special treatment: any serializer type mismatch
during denormalization was unconditionally promoted to HTTP 422. Starting with 4.4, that implicit
promotion is replaced by a constraint-aware check.

**Who is affected**: code that relied on enum-typed properties producing 422 without any Symfony
Validator constraint (or Laravel rule) on the property.

**What to do (Symfony)**: add an explicit constraint on the enum property:

```php
use Symfony\Component\Validator\Constraints as Assert;

#[Assert\Type(Status::class)]
public Status $status;
```

Alternatively, enable Symfony Validator's
[auto-mapping](https://symfony.com/doc/current/validation/auto_mapping.html) on the resource class.
Auto-mapping generates an implicit `Type` constraint from the PHP type declaration, which is
sufficient for the 422 promotion to apply.

**What to do (Laravel)**: add a rule for the property in `rules`:

```php
#[ApiResource(
    rules: ['status' => 'required']
)]
```

Properties that already carry any constraint or rule are unaffected — they continue to produce 422.

For the full rule tables and additional details, see
[Constraint-Aware 422 for Denormalization Errors](../symfony/validation.md#constraint-aware-422-for-denormalization-errors)
(Symfony) and the equivalent section in the
[Laravel validation guide](../laravel/validation.md#constraint-aware-422-for-denormalization-errors).

### Deprecations

#### Legacy Doctrine Filters

The legacy Doctrine filter API is deprecated in favor of parameter-based filters declared with the
`#[QueryParameter]` attribute. The `#[ApiFilter]` attribute, the `Operation::$filters` property, and
the `AbstractFilter` base class (Doctrine ORM and MongoDB ODM) are all deprecated and removed in
6.0.

There are two kinds of migration:

- **Replaced filters** — removed in 6.0, swap the class:

    | Legacy filter                                        | Replacement                                                                     |
    | ---------------------------------------------------- | ------------------------------------------------------------------------------- |
    | `SearchFilter`                                       | `ExactFilter` / `PartialSearchFilter` / `IriFilter` (depending on the strategy) |
    | `BooleanFilter`, `NumericFilter`, `BackedEnumFilter` | `ExactFilter`                                                                   |
    | `OrderFilter`                                        | `SortFilter`                                                                    |

- **Kept filters** — `DateFilter`, `RangeFilter` and `ExistsFilter` survive. Only the way you
  _declare_ them is deprecated: move the declaration from `#[ApiFilter]` to `#[QueryParameter]`. The
  class name and the URL syntax stay the same (drop-in).

`ComparisonFilter` and `OrFilter` are now stable (no longer experimental) and are the recommended
building blocks for comparison and disjunction filtering.

A codemod automates the rewrite of `#[ApiFilter]` declarations to `#[QueryParameter]`:

```console
bin/console api:upgrade-filter
```

See the [filter migration guide](doctrine-filters.md#migrating-from-apifilter-to-queryparameter) for
the full table and before/after examples.

> [!TIP] When instantiating a filter inside a `QueryParameter`, always use named arguments
> (`new DateFilter(nullManagement: ...)` rather than positional). Filter constructors are refined
> across versions; named arguments keep your declarations forward-compatible.

#### JSON:API `use_iri_as_id`

Not setting `api_platform.jsonapi.use_iri_as_id` explicitly is deprecated. The default changes from
`true` to `false` in 5.0. Set it explicitly to silence the deprecation and lock in the behavior you
want:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    jsonapi:
        use_iri_as_id: true # keep IRIs as the "id" field; set to false to use entity identifiers
```

#### Security `AccessDeniedException`

`ApiPlatform\Symfony\Security\Exception\AccessDeniedException` is deprecated. Use
`ApiPlatform\Metadata\Exception\AccessDeniedException` instead.

#### Denormalization Moved Out of the Item Normalizers

The item normalizers no longer denormalize. Calling `denormalize()` on one of them triggers a
deprecation, and the denormalization code moves to a new `ItemDenormalizer` class in the same
namespace:

| Deprecated denormalization entry point          | Replacement                                       |
| ----------------------------------------------- | ------------------------------------------------- |
| `ApiPlatform\Serializer\ItemNormalizer`         | `ApiPlatform\Serializer\ItemDenormalizer`         |
| `ApiPlatform\JsonLd\Serializer\ItemNormalizer`  | `ApiPlatform\JsonLd\Serializer\ItemDenormalizer`  |
| `ApiPlatform\JsonApi\Serializer\ItemNormalizer` | `ApiPlatform\JsonApi\Serializer\ItemDenormalizer` |
| `ApiPlatform\GraphQl\Serializer\ItemNormalizer` | `ApiPlatform\GraphQl\Serializer\ItemDenormalizer` |

This affects you only if you decorate or extend one of these classes to change how API Platform
reads an incoming payload. If you decorate a normalizer service to alter denormalization, decorate
the matching denormalizer service instead:

| Format   | Denormalizer service                        |
| -------- | ------------------------------------------- |
| Default  | `api_platform.serializer.denormalizer.item` |
| JSON-LD  | `api_platform.jsonld.denormalizer.item`     |
| JSON:API | `api_platform.jsonapi.denormalizer.item`    |
| GraphQL  | `api_platform.graphql.denormalizer.item`    |

Decoration that only changes normalization keeps working without a change.

## API Platform 4.2 to 4.3

### Breaking Changes

#### Doctrine Filters Require Explicit `property`

Doctrine parameter-based filters (`ExactFilter`, `IriFilter`, `PartialSearchFilter`, `UuidFilter`)
now throw `InvalidArgumentException` if the `property` attribute is missing. If you have filter
parameters without an explicit `property`, you must either add one or use the `:property`
placeholder in your parameter name.

```php
// Before (would silently work without property):
#[ApiFilter(ExactFilter::class)]

// After (property is required):
#[ApiFilter(ExactFilter::class, property: 'name')]
// Or use the :property placeholder in the parameter name
```

#### Readonly Doctrine Entities Lose PUT & PATCH

Entities marked as readonly via Doctrine metadata (`$classMetadata->markReadOnly()`) no longer
expose PUT and PATCH operations. Clients sending PUT/PATCH to these resources will receive a 404. If
you need write operations on readonly entities, explicitly define them in your `ApiResource`
attribute.

#### JSON-LD `@type` with `output` and `itemUriTemplate`

When using `output` with `itemUriTemplate` on a collection operation, the JSON-LD `@type` now uses
the resource class name instead of the output DTO class name for semantic consistency with
`itemUriTemplate` behavior. Update any client code that relies on the DTO class name in `@type`.

### Behavioral Changes

#### `isGranted` Evaluated Before Provider

Security expressions are now evaluated before the state provider runs. Expressions that do not
reference the `object` variable will be checked at the `pre_read` stage, improving security by
preventing unnecessary database queries on unauthorized requests. Expressions that reference
`object` still wait for the provider to resolve the entity. Review any security expressions that
relied on provider side-effects running before authorization.

#### Hydra Class `@id` Now Always Uses `#ShortName`

Hydra documentation classes now consistently use `#ShortName` as their `@id` instead of schema.org
type URIs (e.g. `schema:Product`). Semantic types configured via `types` are now exposed through
`rdfs:subClassOf`. Clients should expect class `@id` and property range changes in the Hydra
documentation if resources had custom `types` configured.

#### LDP-Compliant Response Headers

API responses now include `Allow` and `Accept-Post` headers per the Linked Data Platform
specification. These are informational headers that help clients discover API capabilities and
should not break existing integrations.

## API Platform 3.4

Remove the `keep_legacy_inflector`, the `event_listeners_backward_compatibility_layer` and the
`rfc_7807_compliant_errors` flag:

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

Indeed, we will throw another validation class in API Platform 4 we will throw
`ApiPlatform\Validator\Exception\ValidationException` instead of
`ApiPlatform\Symfony\Validator\Exception\ValidationException`

It's really important to add the `use_symfony_listeners` flag, set to `true` if you use Symfony
listeners or controllers:

```yaml
api_platform:
    use_symfony_listeners: false
```

The `keep_legacy_inflector` flag will be removed from API Platform 4, you need to fix your issues
first. In API Platform 3.4, the Inflector is available as a service that you can configure through:

```yaml
api_platform:
    inflector: api_platform.metadata.inflector
```

Implement the `ApiPlatform\Metadata\InflectorInterface` if you need to tweak its behavior.

We added an `hydra_prefix` configuration as the `hydra:` prefix will be removed by default in API
Platform 4:

```yaml
api_platform:
    serializer:
        hydra_prefix: false
```

Standard PUT is now `true` by default, you can change its value using:

```yaml
api_platform:
    defaults:
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

This is the recommended configuration for API Platform 3.2. We review each of these changes in this
document.

```yaml
api_platform:
    title: Hello API Platform
    version: 1.0.0
    formats:
        jsonld: ["application/ld+json"]
    docs_formats:
        jsonld: ["application/ld+json"]
        jsonopenapi: ["application/vnd.openapi+json"]
        html: ["text/html"]
    defaults:
        stateless: true
        cache_headers:
            vary: ["Content-Type", "Authorization", "Origin"]
        extra_properties:
            standard_put: true
            rfc_7807_compliant_errors: true
    event_listeners_backward_compatibility_layer: false
    keep_legacy_inflector: false
```

### Formats

We noticed that API Platform was enabling `json` by default because of our OpenAPI support. We
introduced the new `application/vnd.openapi+json`. Therefore if you want `json` you need to
explicitly handle it:

```yaml
formats:
    json: ["application/json"]
```

You can also remove documentations you're not using via the new `docs_formats`.

A new option `error_formats` is also used for content negotiation.

### Event listeners

For new users we recommend to use

```yaml
event_listeners_backward_compatibility_layer: false
```

This allows API Platform to not use http kernel event listeners. It also allows you to force options
like `read: true` or `validate: true`. This simplifies use cases like
[validating a delete operation](https://api-platform.com/docs/v3.2/guides/delete-operation-with-validation/)
Event listeners will not get removed and are not deprecated, they'll use our providers and
processors in a future version.

### Inflector

We're switching to `symfony/string`
[inflector](https://symfony.com/doc/current/components/string.html#inflector), to keep using
`doctrine/inflector` use:

```yaml
keep_legacy_inflector: true
```

We strongly recommend that you use your own inflector anyways with a
[PathSegmentNameGenerator](https://github.com/api-platform/core/blob/f776f11fd23e5397a65c1355a9ebcbb20afac9c2/src/Metadata/Operation/UnderscorePathSegmentNameGenerator.php).

### Errors

```yaml
defaults:
    extra_properties:
        rfc_7807_compliant_errors: true
```

As this is an `extraProperties` it's configurable per resource/operation. This is improving the
compatibility of Hydra errors with JSON problem. It also enables new extension points on
[Errors](https://api-platform.com/docs/v3.2/core/errors/) such as
[Error provider](https://api-platform.com/docs/v3.2/guides/error-provider/) and
[Error Resource](https://api-platform.com/docs/v3.2/guides/error-resource/).

### OpenApi context

You may want to convert your openApiContext to openapi, doing so is quite fastidious, @lyrixx
created a rector script to help if needed:

[https://github.com/lyrixx/rector-apip-openapi](https://github.com/lyrixx/rector-apip-openapi)
