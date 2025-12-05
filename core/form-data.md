# Accept `application/x-www-form-urlencoded` Form Data

API Platform only supports raw documents as request input (encoded in JSON, XML, YAML...). This has many advantages
including support of types and the ability to send back to the API documents originally retrieved through a `GET` request.
However, sometimes - for instance, to support legacy clients - it is necessary to accept inputs encoded in the traditional
[`application/x-www-form-urlencoded`](https://www.w3.org/TR/html401/interact/forms.html#h-17.13.4.1) format
(HTML form content type). This can easily be done using the powerful [System providers and processors](extending.md#system-providers-and-processors)
of the framework.

> [!WARNING]  
> Adding support for `application/x-www-form-urlencoded` makes your API vulnerable to [CSRF (Cross-Site Request Forgery)](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)) attacks.  
> It's crucial to implement proper countermeasures to protect your application.
>
> If you're using Symfony, make sure you enable [Stateless CSRF protection](https://symfony.com/blog/new-in-symfony-7-2-stateless-csrf).  
>
> If you're working with Laravel, refer to the [Laravel CSRF documentation](https://laravel.com/docs/csrf) to ensure
> adequate protection against such attacks.

## Configuration

First, you must register the form format and map it to the `application/x-www-form-urlencoded` MIME type in your API Platform configuration:  

```yaml
# api\_platform.yaml  
api_platform:  
    formats:  
        jsonld: ['application/ld+json']  
        form: ['application/x-www-form-urlencoded']
```

## Creating a Decoder

The Symfony Serializer (used by API Platform) does not decode `application/x-www-form-urlencoded` by default.
You need to create a custom decoder that implements DecoderInterface to handle this format.  

```php
<?php  
// src/Serializer/FormUrlEncodedDecoder.php

namespace App\Serializer;

use Symfony\Component\Serializer\Encoder\DecoderInterface;

class FormUrlEncodedDecoder implements DecoderInterface  
{  
    public function decode(string $data, string $format, array $context = []): mixed  
    {  
        if (\!is_string($data)) {  
            throw new \InvalidArgumentException('Data must be a string.');  
        }

        parse_str($data, $parsedData);

        return $parsedData;  
    }

    public function supportsDecoding(string $format): bool  
    {  
        return $format === 'form';  
    }  
}
```

## **Usage in a Resource**

You can now configure your API Resource to accept the form input format. In this example, we define a FormData resource and restrict the inputFormats for the Post operation.  

```php
<?php  
// src/ApiResource/FormData.php

namespace App\ApiResource;

use ApiPlatform\Metadata\Operation;  
use ApiPlatform\Metadata\Post;

#[Post(  
    inputFormats: ['form' => ['application/x-www-form-urlencoded']],  
    processor: [self::class, 'process']  
)]  
class FormData {  
    public string $name;

    public static function process(mixed $data, Operation $operation, array $uriVariables, array $context) {  
        return $data;  
    }  
}  
```
