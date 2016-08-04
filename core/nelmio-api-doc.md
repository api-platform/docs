# NelmioApiDocBundle Integration

![Screenshot of ApiBundle integrated with NelmioApiDocBundle](../tutorial/images/api-doc.png)

[NelmioApiDocBundle](https://github.com/nelmio/NelmioApiDocBundle) (since version 2.9) has built-in support for API Platform.
Installing it will give you access to a human-readable documentation and a nice sandbox.

If you use the API Platform standard edition, NelmioApiDocBundle is automatically configured. Browse the `/doc` URL to show
it.

If you use the standalone API Platform Core bundle, copy the following configuration:

```yaml
# app/config/config.yml

api_platform:
    # ...
    enable_nelmio_api_doc: true

nelmio_api_doc:
    sandbox:
        accept_type:        'application/json'
        body_format:
            formats:        [ 'json' ]
            default_format: 'json'
        request_format:
            formats:
                json:       'application/json'
```

Please note that NelmioApiDocBundle has a sandbox limitation where you cannot pass a JSON array as parameter, so you cannot
use it to deserialize nested objects.

Previous chapter: [Configuration](configuration.md)
Next chapter: [Operations](operations.md)
