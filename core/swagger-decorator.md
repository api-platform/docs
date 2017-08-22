# Override Swagger documentation


Symfony allow to [decorates services](https://symfony.com/doc/current/service_container/service_decoration.html), here we need to decorate the

`api_platform.swagger.normalizer.documentation`

1. declare the decorator and class

 ```yaml
 # app/config/services.yml
 
 services:
      app.swagger.swagger_decorator:
         decorates: api_platform.swagger.normalizer.documentation
         class: AppBundle\Swagger\SwaggerDecorator
         arguments: ['@app.swagger.swagger_decorator.inner']
 ```

```php
<?php

namespace AppBundle\Swagger;

use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

/**
 * Class SwaggerDecorator.
 */
class SwaggerDecorator implements NormalizerInterface
{
    /**
     * @var NormalizerInterface
     */
    private $decorated;

    public function __construct(NormalizerInterface $decorated)
    {
        $this->decorated = $decorated;
    }

    public function normalize($object, $format = null, array $context = [])
    {
        $docs = $this->decorated->normalize($object, $format, $context);

		// Your code
			
        return $docs;
    }

    public function supportsNormalization($data, $format = null)
    {
        return $this->decorated->supportsNormalization($data, $format);
    }
}
```

2. In Swagger decorator you can acces to the docs

the variable `docs` is an array compose by paths :

`$docs['paths']['{path}']['{method}']['parameters']`

for example if you want to remove all references to your path ’/foos’ in method GET in the swagger you can make in the normalize function

```php
		unset($docs['paths']['/foos']['get']);
```


