# Rate Limiting

You will usually want to add some form of rate limit to your API if it's publicly accessible, to help reduce the risk of
[denial of service attacks](https://en.wikipedia.org/wiki/Denial-of-service_attack).

A simple, but extensible rate limiter is now provided with API Platform, but is disabled by default.


## Turning it on

To enable the rate limiter, simply add the `rate_limits` key to the root of the `api_platform` config in
`./app/config/config.yml` (or whatever other format of config file you're using, and set `enabled` to `true`…

```
api_platform:
    # ...
    rate_limits:
        enabled:        true
        # ...
```


## Selecting a Storage Method

For now the rate limiter uses APCu to track rate limit counts, however in future other cache storage mechanisms may be
added.  To select the storage mechanism you'd like to use, add the `storage_method` key, with the identifier key of the
storage mechanism service you would like to use…

```
api_platform:
    # ...
    rate_limits:
        # ...
        storage_method: apc
        # ...
```

Note: As APCu is local to the container, and rate limits will be specific to each container, so in theory for a limit of
10 request per second, using APCu storage, when you are running 3 containers, a user could possibly make up to 30
requests per second (10 on each container), so bear this in mind when setting the rate limits.  In future, there may be
a centralised store, such as Redis, so you may need to modify your rate limits if you change.


## Limiters & Configuring the Rate Limits

Rate limits are handled by rate limiter services.  These are services that implement the
`\ApiPlatform\Core\RateLimit\RateLimiterInterface`.  Currently there is only 1 rate limiting service provided, to rate
limit per IP address, but more may be added in future.  To implement your own limiter, simply register a new service
that implements the `\ApiPlatform\Core\RateLimit\RateLimiterInterface`, and if you've got auto-configuring enabled, as
per the default api-platform install, then this should automatically be picked up and enabled.  If not, you simply need
to tage the service with the  `api_platform.rate_limiter` tag.

Each configured rate limiter is expected to return a string unique to the type of request being measured, for example
the IP address rate limiter simply returns the requesting IP address as a string.  If you wanted to rate limit request
for each user, or each endpoint then you could add a limiter that returned the user's username or endpoint route name,
or you could combine any number of bits of information for more complext rate limiting.

The window in which each limiter will rate limit also needs to be configured.  To do this, simple add the `limits`
section to the `api_platform.rate_limits` config…

```
api_platform:
    # ...
    rate_limits:
        enabled:        true
        storage_method: apc
        limits:
            per_ip_address:         # <== The identifier defined in the rate limiter service
                time_frame: '10s'   # <== Defines the timing window to perform rate limiting within
                capacity:   20      # <== The number of requests allowed in the given timing window
```

The timing window can be defined for any number of seconds, minutes, hours, days, weeks, months, or even years.  Simply
set the `time_frame` value to a string starting with the number of units, followed by a single character representing
the desired window…

* Y = years
* M = months
* W = weeks
* D = days
* h = hours
* m = minutes
* s = seconds