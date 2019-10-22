# Deploying API Platform Applications

## Cloud Providers
API Platform apps are super-easy to deploy in production in most cloud providers:

* [Kubernetes](kubernetes.md)
* [Heroku](heroku.md)
* [Platform.sh](https://platform.sh/blog/deploy-api-platform-on-platformsh)

## Deploying an API server

The server part of API Platform is basically a standard Symfony application, that you can easily deploy on your own servers:

* [Configure a web server](https://symfony.com/doc/current/setup/web_server_configuration.html) (read the [note below](#using-nginx-along-with-varnish-andor-http-push) if you're using nginx)
* [Deploy a Symfony application](http://symfony.com/doc/current/deployment.html) 
 
### Using nginx along with Varnish and/or HTTP Push:

Using *Varnish* and/or *HTTP Push* with API-Platform can lead the server to send headers with a certain size, which might cause nginx to throw 502 errors with the following message:

> upstream sent too big header while reading response header from upstream

In some cases, the default nginx header size (4k) can be overflown when your API gets larger and more complex.

To avoid this, it is recommended that you increase the [`fastcgi_buffers`](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_buffers) 
and [`fastcgi_buffer_size`](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_buffer_size) values 
according to your needs.

Example config:

```nginx
server {
    server_name domain.tld www.domain.tld;
    root /var/www/api/public;

    location / {
        try_files $uri /index.php$is_args$args;
    }

    location ~ ^/index\.php(/|$) {
        fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;

        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;

        # Increase fastcgi_buffers and fastcgi_buffer_size
        fastcgi_buffer_size 32k;
        fastcgi_buffers 32 4k;

        internal;
    }

    location ~ \.php$ {
        return 404;
    }

    error_log /var/log/nginx/project_error.log;
    access_log /var/log/nginx/project_access.log;
}
```

## Deploying Client Applications

The clients are [Create React App](https://github.com/facebook/create-react-app/) skeletons. You can deploy them in a wink
on any static website hosting service (including [Netlify](https://www.netlify.com/), [Firebase Hosting](https://firebase.google.com/docs/hosting/),
[GitHub Pages](https://pages.github.com/), or [Amazon S3](https://docs.aws.amazon.com/en_us/AmazonS3/latest/dev/WebsiteHosting.html)
by following [the relevant documentation](https://facebook.github.io/create-react-app/docs/deployment).

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/ansistrano?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="JWT screencast"><br>Watch the Animated Deployment with Ansistrano screencast</a></p>
