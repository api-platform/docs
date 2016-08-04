# Performance

## Enabling the Metadata Cache

Computing metadata used by the bundle is a costly operation. Fortunately, metadata can be computed once and then cached.
API Platform internally uses a [PSR-6](http://www.php-fig.org/psr/psr-6/) cache. If the Symfony Cache Component is available
(the default in the standard edition), it automatically enables the support for the best cache adapter available.

Best performance is achieved using [APCu](https://github.com/krakjoe/apcu). Be sure to have the APCu extension installed
on your production server, API Platform will automatically use it.

## Using PPM (PHP-PM)

Response time of the API can be improved up to 15x by using [PHP Process Manager](https://github.com/php-pm/php-pm). If
you want to use it on your project, follow the documentation dedicated to Symfony on the PPM website.

Keep in mind that PPM is still in an early stage of development and can cause issues in production.

Previous chapter: [Security](security.md)<br>
Next chapter: [Path Naming Strategy](path-naming-strategy.md)
