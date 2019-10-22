# ReDoc UI

API-Platform also ships with [ReDoc UI](https://redocly.github.io/redoc/) to vizualize and interact with your resources.

## Changing the Location of ReDoc UI

Sometimes you may want to have the API at one location, and the ReDoc UI at a different location. 
This can be done by disabling the ReDoc UI from the API Platform configuration file and manually adding the documentation UI controller.

### Disabling ReDoc

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    # ...
    enable_re_doc: false
```

### Manually Registering the ReDoc Controller

```yaml
# api/config/routes.yaml
swagger_ui:
    path: /docs
    controller: api_platform.swagger.action.ui
```

Change `/docs` to the URI you wish ReDoc to be accessible on.
