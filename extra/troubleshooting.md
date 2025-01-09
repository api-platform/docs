# Troubleshooting

This is a list of common pitfalls while using API Platform, and how to avoid them.

## Using Docker

### With Docker Toolbox on Windows

Docker Toolbox is not supported anymore by API Platform. Please upgrade to [Docker for Windows](https://www.docker.com/docker-windows).

### Error Starting The Web Server

If the `php` container cannot start and display this `Error starting userland proxy: Bind for 0.0.0.0:80`, it means that port 80 is already in use. You can check to see which processes are currently listening on certain ports.

Find out if any service listens on port 80. You can use this command on UNIX-based OSes like MacOS and Linux:

```console
sudo lsof -n -i :80 | grep LISTEN
```

On Windows, you can use `netstat`. This will give you all TCP/IP network connections and not just processes listening to port 80.

```console
netstat -a -b
```

You can change the port to be used in the `docker-compose.yml` file (default is port 80).

## Using API Platform and JMS Serializer in the same project

For the latest versions of [JMSSerializerBundle](http://jmsyst.com/bundles/JMSSerializerBundle), there is no conflict so everything should work out of the box.

If you are still using the old, unmaintained v1 of JMSSerializerBundle, the best way would be to [upgrade to v2](https://github.com/schmittjoh/JMSSerializerBundle/blob/2.4.2/UPGRADING.md#upgrading-from-1x-to-20) of JMSSerializerBundle.

In v1 of JMSSerializerBundle, the `serializer` alias is registered for the JMS Serializer service by default. However, API Platform requires the Symfony Serializer (and not the JMS one) to work properly. If you cannot upgrade for some reason, this behavior can be deactivated using the following configuration:

```yaml
# config/packages/jms_serializer.yaml
jms_serializer:
    enable_short_alias: false
```

The JMS Serializer service is available as `jms_serializer`.

**Note:** if you are using JMSSerializerBundle along with FOSRestBundle and considering migrating to API Platform, you might want to take a look at [this guide](migrate-from-fosrestbundle.md) too.

## "upstream sent too big header while reading response header from upstream" NGINX 502 Error

Some of your API calls fail with a 502 error and the logs for the api container shows the following error message `upstream sent too big header while reading response header from upstream`.

This can be due to the cache invalidation headers that are too big for NGINX. When you query the API, API Platform adds the ids of all returned entities and their dependencies in the headers like so : `Cache-Tags: /entity/1,/dependent_entity/1,/entity/2`. This can overflow the default header size (4k) when your API gets larger and more complex.

You can modify the `api/docker/nginx/conf.d/default.conf` file and set values to `fastcgi_buffer_size` and `fastcgi_buffers` that suit your needs, like so:

```nginx
server {
    root /srv/api/public;

    location / {
        # try to serve file directly, fallback to index.php
        try_files $uri /index.php$is_args$args;
    }

    location ~ ^/index\.php(/|$) {
        # Comment the next line and uncomment the next to enable dynamic resolution (incompatible with Kubernetes)
        fastcgi_pass php:9000;
        #resolver 127.0.0.11;
        #set $upstream_host php;
        #fastcgi_pass $upstream_host:9000;
        
        # Bigger buffer size to handle cache invalidation headers expansion
        fastcgi_buffer_size 32k;
        fastcgi_buffers 8 16k;

        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        # When you are using symlinks to link the document root to the
        # current version of your application, you should pass the real
        # application path instead of the path to the symlink to PHP
        # FPM.
        # Otherwise, PHP's OPcache may not properly detect changes to
        # your PHP files (see https://github.com/zendtech/ZendOptimizerPlus/issues/126
        # for more information).
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;
        # Prevents URIs that include the front controller. This will 404:
        # http://domain.tld/index.php/some-path
        # Remove the internal directive to allow URIs like this
        internal;
    }

    # return 404 for all other php files not matching the front controller
    # this prevents access to other php files you don't want to be accessible.
    location ~ \.php$ {
      return 404;
    }
}
```
