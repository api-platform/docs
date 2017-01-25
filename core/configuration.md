# Configuration

Here's the complete configuration of the Symfony bundle with including default values:

```yaml
# app/config/config.yml

api_platform:

    # The title of the API.
    title: ''

    # The description of the API.
    description: ''

    # The version of the API.
    version: '0.0.0'

    # Specify the default operation path resolver to use for generating resources operations path.
    default_operation_path_resolver: 'api_platform.operation_path_resolver.underscore'

    # Specify a name converter to use.
    name_converter: ~

    eager_loading:
        # To enable or disable eager loading.
        enabled: true

        # Max number of joined relations before EagerLoading throws a RuntimeException.
        max_joins: 30

        # Force join on every relation. If disabled, it will only join relations having the EAGER fetch mode.
        force_eager: true

    # Enable the FOSUserBundle integration.
    enable_fos_user: false

    # Enable the Nelmio Api doc integration.
    enable_nelmio_api_doc: false

    # Enable the Swagger documentation and export.
    enable_swagger: true

    # Enable Swagger ui.
    enable_swagger_ui: true

    collection:
        # The default order of results.
        order: ~

        # The name of the query parameter to order results.
        order_parameter_name: 'order'

        pagination:
            # To enable or disable pagination for all resource collections by default.
            enabled: true

            # To allow the client to enable or disable the pagination.
            client_enabled: false

            # To allow the client to set the number of items per page.
            client_items_per_page: false

            # The default number of items per page.
            items_per_page: 30

            # The maximum number of items per page.
            maximum_items_per_page: ~

            # The default name of the parameter handling the page number.
            page_parameter_name: 'page'

            # The name of the query parameter to enable or disable pagination.
            enabled_parameter_name: 'pagination'

            # The name of the query parameter to set the number of items per page.
            items_per_page_parameter_name: 'itemsPerPage'

    # The list of exceptions mapped to their HTTP status code.
    exception_to_status:
        # With a status code.
        Symfony\Component\Serializer\Exception\ExceptionInterface: 400

        # Or with a constant defined in the 'Symfony\Component\HttpFoundation\Response' class.
        ApiPlatform\Core\Exception\InvalidArgumentException: 'HTTP_BAD_REQUEST'

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

    # The list of enabled error formats. The first one will be the default.
    error_formats:
        jsonproblem:
            mime_types: ['application/problem+json']
            
        jsonld:
            mime_types: ['application/ld+json']
            
        # ...
     
     # Custom paths to look for api resource configurations
     loader_paths:
         annotation: []
         xml: []
         yaml:
            - "%kernel.root_dir%/../src/api/user_resources.yml"
```

Previous chapter: [Getting Started](getting-started.md)

Next chapter: [Operations](operations.md)
