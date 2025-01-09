# NelmioApiDocBundle Integration

NelmioApiDoc provides an alternative to [the native Swagger/Open API support](swagger.md) provided by API Platform.

As NelmioApiDocBundle 3+ has built-in support for API Platform, this documentation is only relevant for people using
NelmioApiDocBundle between version 2.9 and 3.0.

For new projects, prefer using the built-in Swagger support and/or NelmioApiDoc 3.

![Screenshot of API Platform integrated with NelmioApiDocBundle](images/NelmioApiDocBundle.png)

[NelmioApiDocBundle](https://github.com/nelmio/NelmioApiDocBundle) is supported by API Platform since version 2.9.

To enable the NelmioApiDoc integration, copy the following configuration:

```yaml
# config/packages/api_platform.yaml
api_platform:
    # ...

    enable_nelmio_api_doc: true

nelmio_api_doc:
    sandbox:
        accept_type: 'application/json'
        body_format:
            formats: ['json']
            default_format: 'json'
        request_format:
            formats:
                json: 'application/json'
```

Please note that NelmioApiDocBundle has a sandbox limitation where you cannot pass a JSON array as parameter, so you cannot
use it to deserialize nested objects.
