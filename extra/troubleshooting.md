# Troubleshooting

This is a list of common pitfalls on using API Platform, and how to avoid them.

## Using Docker

### With Docker Toolbox on Windows

Docker Toolbox is not supported anymore by API Platform. Please upgrade to [Docker for Windows](https://www.docker.com/docker-windows).

### Error starting userland proxy

If the `app` container cannot start and display this `Error starting userland proxy: Bind for 0.0.0.0:80`, it means that port 80 is already in use. You can check to see which processes are currently listening on certain ports.

```bash
# Find out if any service listens on port 80.
# You can use this command on UNIX-based OSes like MacOS and Linux.
sudo lsof -n -i :80 | grep LISTEN

# For Windows, you can use netstat. 
# This will give you all TCP/IP network connections and not just processes listening to port 80.
netstat -a -b
```

You can change the port to be used in the `docker-compose.yml` file (default is port 80).

## Using API Platform and JMS Serializer in the same project

By default, [JMS Serializer Bundle](http://jmsyst.com/bundles/JMSSerializerBundle) replaces the `serializer` service by its own. However, API Platform requires the Symfony serializer (and not the JMS one) to work properly.
Fortunately, this behavior can be deactivated using the following configuration:

```yaml
# api/config/packages/api_platform.yaml

jms_serializer:    
    enable_short_alias: false
```
