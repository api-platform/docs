# Troubleshooting

This is a list of common pitfalls on using API Platform, and how to avoid them.

## Using Docker

### With Docker Toolbox on Windows

If you get errors like the following when running `docker-compose up` on Windows:

```
ERROR: for app  Cannot create container for service app: Invalid bind mount spec "C:\\Users\\Kevin\\api-platform:/srv/api-platform:rw": Invalid volume specification: 'C:\Users\Kevin\api-platform:/srv/api-platform:rw'
←[31mERROR←[0m: Encountered errors while bringing up the project.
```

Be sure to set the `COMPOSE_CONVERT_WINDOWS_PATHS` environment variable to `1`. It can be done by creating a file called `.env` in the root of the project containing this line:

```
COMPOSE_CONVERT_WINDOWS_PATHS=1
```

### Error starting userland proxy

If the `app` container cannot start and display this `Error starting userland proxy: Bind for 0.0.0.0:80`, it means that port 80 is already in use.

You can change the port to be used in the `docker-compose.yml` file (default is port 80).

## Using API Platform and JMS Serializer in the same project

By default, [JMS Serializer](http://jmsyst.com/bundles/JMSSerializerBundle) replaces the `serializer` service by its own. However, API Platform requires the Symfony serializer (and not the JMS one) to work properly.
Fortunately, this behavior can be deactivated using the following configuration:

```yaml
# app/config/config.yml

jms_serializer:    
    enable_short_alias: false
```
