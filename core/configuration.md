# Configuration

Here's the complete configuration of the Symfony bundle including default values:

```yaml
# api/config/packages/api_platform.yaml
api_platform:

    # The title of the API.
    title: 'API title'

    # The description of the API.
    description: 'API description'

    # The version of the API.
    version: '0.0.0'

    # Set this to false if you want Webby to disappear.
    show_webby: true

    # Specify a name converter to use.
    name_converter: ~

    # Specify an asset package name to use.
    asset_package: null

    # Specify a path name generator to use.
    path_segment_name_generator: 'api_platform.path_segment_name_generator.underscore'

    validator:
        # Enable the serialization of payload fields when a validation error is thrown.
        # If you want to serialize only some payload fields, define them like this: [ severity, anotherPayloadField ]
        serialize_payload_fields: []

        # To enable or disable query parameters validation on collection GET requests
        query_parameter_validation: true

    eager_loading:
        # To enable or disable eager loading.
        enabled: true

        # Fetch only partial data according to serialization groups.
        # If enabled, Doctrine ORM entities will not work as expected if any of the other fields are used.
        fetch_partial: false

        # Max number of joined relations before EagerLoading throws a RuntimeException.
        max_joins: 30

        # Force join on every relation.
        # If disabled, it will only join relations having the EAGER fetch mode.
        force_eager: true

    # Enable the Swagger documentation and export.
    enable_swagger: true

    # Enable Swagger UI.
    enable_swagger_ui: true

    # Enable ReDoc.
    enable_re_doc: true

    # Enable the entrypoint.
    enable_entrypoint: true

    # Enable the docs.
    enable_docs: true

    # Enable the data collector and the WebProfilerBundle integration.
    enable_profiler: true

    # Use Symfony event listeners instead of providers and processors.
    event_listeners_backward_compatibility_layer: false

    # Keep doctrine/inflector instead of symfony/string to generate plurals for routes.
    keep_legacy_inflector: false

    collection:
        # The name of the query parameter to filter nullable results (with the ExistsFilter).
        exists_parameter_name: 'exists'

        # The default order of results.
        order: 'ASC'

        # The name of the query parameter to order results (with the OrderFilter).
        order_parameter_name: 'order'

        pagination:
            # The default name of the parameter handling the page number.
            page_parameter_name: 'page'

            # The name of the query parameter to enable or disable pagination.
            enabled_parameter_name: 'pagination'

            # The name of the query parameter to set the number of items per page.
            items_per_page_parameter_name: 'itemsPerPage'

            # The name of the query parameter to enable or disable the partial pagination.
            partial_parameter_name: 'partial'

    mapping:
        # The list of paths with files or directories where the bundle will look for resource files (if you use this, you need to add the current project path and the new one)
        paths: ["%kernel.project_dir%/src", "%kernel.project_dir%/vendor/foo-vendor/foo-bundle/src"]

    # The list of your resources class directories. Defaults to the directories of the mapping paths but might differ.
    resource_class_directories:
        - '%kernel.project_dir%/src/Entity'

    doctrine:
        # To enable or disable Doctrine ORM support.
        enabled: true

    doctrine_mongodb_odm:
        # To enable or disable Doctrine MongoDB ODM support.
        enabled: false

    oauth:
        # To enable or disable OAuth.
        enabled: false

        # The OAuth client ID.
        clientId: ''

        # The OAuth client secret.
        clientSecret: ''

        # The OAuth type.
        type: 'oauth2'

        # The OAuth flow grant type.
        flow: 'application'

        # The OAuth token URL. Make sure to check the specification tokenUrl is not needed for an implicit flow.
        tokenUrl: ''

        # The OAuth authentication URL.
        authorizationUrl: ''

        # The OAuth scopes.
        scopes: []

    graphql:
        # Enabled by default with installed webonyx/graphql-php.
        enabled: false

        # The default IDE (graphiql or graphql-playground) used when going to the GraphQL endpoint. False to disable.
        default_ide: 'graphiql'

        graphiql:
            # Enabled by default with installed webonyx/graphql-php and Twig.
            enabled: false

        graphql_playground:
            # Enabled by default with installed webonyx/graphql-php and Twig.
            enabled: false

        introspection:
            # Enabled by default with installed webonyx/graphql-php.
            enabled: true

        # The nesting separator used in the filter names.
        nesting_separator: _

        collection:
            pagination:
                enabled: true

    swagger:
        # The active versions of OpenAPI to be exported or used in the swagger_ui. The first value is the default.
        versions: [2, 3]

        # The swagger API keys.
        api_keys: []
            # The name of the header or query parameter containing the API key.
            # name: ''

            # Whether the API key should be a query parameter or a header.
            # type: 'query' or 'header'

        swagger_ui_extra_configuration:
            # Controls the default expansion setting for the operations and tags. It can be 'list' (expands only the tags), 'full' (expands the tags and operations) or 'none' (expands nothing).
            docExpansion: list
            # If set, enables filtering. The top bar will show an edit box that you can use to filter the tagged operations that are shown.
            filter: false
            # You can use any other configuration parameters too.

    openapi:
        # The contact information for the exposed API.
        contact:
            # The identifying name of the contact person/organization.
            name:
            # The URL pointing to the contact information. MUST be in the format of a URL.
            url:
            # The email address of the contact person/organization. MUST be in the format of an email address.
            email:
        # A URL to the Terms of Service for the API. MUST be in the format of a URL.
        termsOfService:
        # The license information for the exposed API.
        license:
            # The license name used for the API.
            name:
            # URL to the license used for the API. MUST be in the format of a URL.
            url:

        swagger_ui_extra_configuration:
            # Controls the default expansion setting for the operations and tags. It can be 'list' (expands only the tags), 'full' (expands the tags and operations) or 'none' (expands nothing).
            docExpansion: list
            # If set, enables filtering. The top bar will show an edit box that you can use to filter the tagged operations that are shown.
            filter: false
            # You can use any other configuration parameters too.

    http_cache:
        # To make all responses public by default.
        public: ~

        invalidation:
            # To enable the tags-based cache invalidation system.
            enabled: false

            # URLs of the Varnish servers to purge using cache tags when a resource is updated.
            varnish_urls: []

            # To pass options to the client charged with the request.
            request_options: []

            # Use another service as the purger for example "api_platform.http_cache.purger.varnish.xkey"
            purger: 'api_platform.http_cache.purger.varnish.ban'

    mercure:
        # Enabled by default with installed symfony/mercure-bundle.
        enabled: false

        # The URL sent in the Link HTTP header. If not set, will default to MercureBundle's default hub URL.
        hub_url: null

    messenger:
        # Enabled by default with installed symfony/messenger and not installed symfony/symfony.
        enabled: false

    elasticsearch:
        # To enable or disable Elasticsearch support.
        enabled: false

        # The hosts to the Elasticsearch nodes.
        hosts: []

        # The mapping between resource classes and indexes.
        mapping: []

    # The list of exceptions mapped to their HTTP status code.
    exception_to_status:
        # With a status code.
        Symfony\Component\Serializer\Exception\ExceptionInterface: 400

        # Or with a constant defined in the 'Symfony\Component\HttpFoundation\Response' class.
        ApiPlatform\Exception\InvalidArgumentException: !php/const Symfony\Component\HttpFoundation\Response::HTTP_BAD_REQUEST

        ApiPlatform\Exception\FilterValidationException: 400

        Doctrine\ORM\OptimisticLockException: 409

        # ...

    # The list of enabled formats. The first one will be the default.
    formats:
        jsonld:
            mime_types: ['application/ld+json']

        json:
            mime_types: ['application/json']

        html:
            mime_types: ['text/html']

        # ...

    # The list of enabled patch formats. The first one will be the default.
    patch_formats: []

    # The list of enabled error formats. The first one will be the default.
    error_formats:
        jsonproblem:
            mime_types: ['application/problem+json']

        jsonld:
            mime_types: ['application/ld+json']

        # ...

    # Global resources defaults, see in the next section.
    defaults:
        # ...
```

## Global Resources Defaults

If you need to globally configure all the resources instead of adding configuration in each one, it's possible to do so with the `defaults` key:

```yaml
# api/config/packages/api_platform.yaml
api_platform:

    defaults:
        description: ~
        iri: ~
        short_name: ~
        item_operations: ~
        collection_operations: ~

        graphql: ~

        elasticsearch: ~

        security: ~
        security_message: ~
        security_post_denormalize: ~
        security_post_denormalize_message: ~

        cache_headers:
            # Automatically generate etags for API responses.
            etag: true

            # Default value for the response max age.
            max_age: 3600

            # Default value for the response shared (proxy) max age.
            shared_max_age: 3600

            # Default values of the "Vary" HTTP header.
            vary: ['Accept']

            invalidation:
                xkey:
                    glue: ' '

        normalization_context:
            # Default value to omit null values in conformance with the JSON Merge Patch RFC.
            skip_null_values: true
        denormalization_context: ~
        swagger_context: ~
        openapi_context: ~
        deprecation_reason: ~
        fetch_partial: ~
        force_eager: ~
        formats: ~
        filters: ~
        hydra_context: ~
        mercure: ~
        messenger: ~
        order: ~

        # To enable or disable pagination for all resource collections.
        pagination_enabled: true

        # To allow the client to enable or disable the pagination.
        pagination_client_enabled: false

        # To allow the client to set the number of items per page.
        pagination_client_items_per_page: false

        # To allow the client to enable or disable the partial pagination.
        pagination_client_partial: false

        # The default number of items per page.
        pagination_items_per_page: 30

        # The maximum number of items per page.
        pagination_maximum_items_per_page: ~

        # To allow partial pagination for all resource collections.
        # This improves performances by skipping the `COUNT` query.
        pagination_partial: false

        # To use cursor-based pagination.
        pagination_via_cursor: ~

        pagination_fetch_join_collection: ~

        route_prefix: ~
        validation_groups: ~
        sunset: ~
        input: ~
        output: ~
        stateless: ~

        # The URL generation strategy to use for IRIs
        url_generation_strategy: !php/const ApiPlatform\Api\UrlGeneratorInterface::ABS_PATH

        # To enable collecting denormalization errors
        collectDenormalizationErrors: false

        # ...
```
