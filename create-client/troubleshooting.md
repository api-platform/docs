# Troubleshooting

## Self-Signed TLS Certificate

If you are running API Platform on development machine which does not have valid TLS certificate,
add `NODE_TLS_REJECT_UNAUTHORIZED=0` before running create-client:

```console
NODE_TLS_REJECT_UNAUTHORIZED=0 npm init @api-platform/client --generator typescript https://127.0.0.1:8000/api src/
```

## Authenticated API

The generator does not perform any authentication, so you must ensure that all referenced Hydra paths for your API are
accessible anonymously. If you are using API Platform this will at least include:

```console
api_entrypoint                             ANY      ANY      ANY    /{index}.{_format}
api_doc                                    ANY      ANY      ANY    /docs.{_format}
api_jsonld_context                         ANY      ANY      ANY    /contexts/{shortName}.{_format}
```

For instance, the `access_control` definition in the `security.yaml` could look like this:
```yaml
access_control:
    - { path: ^/index.jsonld, roles: PUBLIC_ACCESS }
    - { path: ^/docs.jsonld, roles: PUBLIC_ACCESS }
    - { path: ^/contexts/hydra.jsonld, roles: PUBLIC_ACCESS }
    - ...
```

You can also define a `--bearer` arg in case of [JWT authentication](../core/jwt.md), or the `--username` and `--password` arguments in case of basic authentication:
```console
docker compose exec pwa \
    pnpm create @api-platform/client --resource book -g next --bearer eyJ0eXAiOiJKV1QiLCJhbGciO...
```

## ApiDocumentation doesn't exist

If you receive `Error: The class http://www.w3.org/ns/hydra/core#ApiDocumentation doesn't exist.` you may have
specified the documentation URL instead of the entrypoint. For example if you are using API Platform and your
documentation URL is at [https://demo.api-platform.com/docs](https://demo.api-platform.com/docs) the entry point is
likely at [https://demo.api-platform.com](https://demo.api-platform.com). You can see an example of the expected
response from an entrypoint in your browser by visiting
[https://demo.api-platform.com/index.jsonld](https://demo.api-platform.com/index.jsonld).

## Cannot read property '@type'

If you receive `TypeError: Cannot read property '@type' of undefined` or `TypeError: Cannot read property '0'
of undefined` check that the URL you specified is accessible and returns jsonld. You can check from the command line
you are using by running something like `curl https://demo.api-platform.com/`.

## Dereferencing a URL did not result in a JSON object

If you receive a message like this:

```console
{ Error
at done (/usr/local/share/.config/yarn/global/node_modules/jsonld/js/jsonld.js:6851:19)
at <anonymous>
at process._tickCallback (internal/process/next_tick.js:188:7)
name: 'jsonld.InvalidUrl',
message: 'Dereferencing a URL did not result in a JSON object. The response was valid JSON, but it was not a JSON object.',
details:
{ code: 'invalid remote context',
url: 'https://demo.api-platform.com/contexts/Entrypoint',
cause: null } }
```

Check access to the specified URL, in this case `https://demo.api-platform.com/contexts/Entrypoint`, use `curl` to check
access and the response `curl https://demo.api-platform.com/contexts/Entrypoint`.

For instance, your API is protected by a JWT strategy, so in the above case an "Access Denied" message in JSON format was being returned. To fix this, you can set the `--bearer` flag with a valid token.

## Docker distribution on Windows and hot-reloading

Due to [a long-time known Docker for Windows issue](https://forums.docker.com/t/file-system-watch-does-not-work-with-mounted-volumes/12038),
the file changes on the host are not notified on the `pwa` container.
It causes the hot-reloading feature to not working properly for Windows users.
