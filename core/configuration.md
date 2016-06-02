# Configuration

Here's the complete configuration of the Symfony bundle with including default values:

```yaml
# app/config/config.yml

api_platform:

    # The title of the API.
    title:                ~ # Required

    # The description of the API.
    description:          ~ # Required

    # The list of enabled formats. The first one will be the default.
    supported_formats:

        # Prototype
        format:
            mime_types:           []

    # Specify a name converter to use.
    name_converter:       null

    # Enable the FOSUserBundle integration.
    enable_fos_user:      false

    # Enable the Nelmio Api doc integration.
    enable_nelmio_api_doc:  true
    collection:

        # The default order of results.
        order:                null

        # The name of the query parameter to order results.
        order_parameter_name:  order
        pagination:

            # To enable or disable pagination for all resource collections by default.
            enabled:              true

            # To allow the client to enable or disable the pagination.
            client_enabled:       false

            # To allow the client to set the number of items per page.
            client_items_per_page:  false

            # The default number of items per page.
            items_per_page:       30

            # The default name of the parameter handling the page number.
            page_parameter_name:  page

            # The name of the query parameter to enable or disable pagination.
            enabled_parameter_name:  pagination

            # The name of the query parameter to set the number of items per page.
            items_per_page_parameter_name:  itemsPerPage
    metadata:
        resource:

            # Cache service for resource metadata.
            cache:                api_platform.metadata.resource.cache.array
        property:

            # Cache service for property metadata.
            cache:                api_platform.metadata.property.cache.array
```

Previous chapter: [Getting Started](getting-started.md)<br>
Next chapter: [NelmioApiDocBundle integration](nelmio-api-doc.md)
