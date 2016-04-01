# NelmioApiDocBundle integration

![Screenshot of ApiBundle integrated with NelmioApiDocBundle](../getting-started/images/api-doc.png)

[NelmioApiDocBundle](https://github.com/nelmio/NelmioApiDocBundle) (since version 2.9) has built-in support for ApiPlatformBundle.
Installing it will give you access to a human-readable documentation and a nice sandbox.

Once installed, use the following configuration:

```yaml

# app/config/config.yml

nelmio_api_doc:
    sandbox:
        accept_type:        "application/json"
        body_format:
            formats:        [ "json" ]
            default_format: "json"
        request_format:
            formats:
                json:       "application/json"
```

Please note that NelmioApiDocBundle has a sandbox limitation where you cannot pass a JSON array as parameter, so you cannot use it to deserialize nested objects.

Previous chapter: [Getting Started](getting-started.md)<br>
Next chapter: [Operations](operations.md)
