# Troubleshooting

This is a list of common pitfalls on using API Platform, and how to avoid them.

## Using Docker

If the web container cannot start and display this `Error starting userland proxy: Bind for 0.0.0.0:80`, it means that the port 80 is already used.

The configuration docker is planned to be launch on the port 80 in the `docker-compose.yml` file.

## Using API Platform and JMS Serializer in the same project

By default, [JMS Serializer](http://jmsyst.com/bundles/JMSSerializerBundle) replaces the `serializer` service by its own. However, API Platform requires the Symfony serializer (and not the JMS one) to works properly.
Fortunately, this behavior can be deactivated using the following configuration:

```yaml
# app/config/config.yml

jms_serializer:    
    enable_short_alias: false
```
