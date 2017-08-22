# Swagger support

## Override Swagger documentation

Symfony allows to [decorate services](https://symfony.com/doc/current/service_container/service_decoration.html), here we need to decorate the

`api_platform.swagger.normalizer.documentation`

### Example

In the following example, we will see how to override Swagger title and add a custom filter in `/foos` path only for the `GET` method 


```yaml
 # app/config/services.yml
 
services:
  app.swagger.swagger_decorator:
     decorates: api_platform.swagger.normalizer.documentation
     class: 'AppBundle\Swagger\SwaggerDecorator'
     arguments: ['@app.swagger.swagger_decorator.inner']
```

```php
<?php

// src\AppBundle\Swagger\SwaggerDecorator.php

namespace AppBundle\Swagger;

use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

final class SwaggerDecorator implements NormalizerInterface
{

    private $decorated;

    public function __construct(NormalizerInterface $decorated)
    {
        $this->decorated = $decorated;
    }

    public function normalize($object, $format = null, array $context = [])
    {
        $docs = $this->decorated->normalize($object, $format, $context);

        $customDefinition = [
            'name' => 'fields',
            'definition' => 'Fields to remove of the outpout',
            'default' => 'id',
            'in' => 'query',
        ];

		
		// e.g add a custom parameter 
		$docs['paths']['/foos']['get']['parameters'][] = $customDefinition;
		
		// Override title
		$docs['info']['title'] = 'My Api Foo';
			
        return $docs;
    }

    public function supportsNormalization($data, $format = null)
    {
        return $this->decorated->supportsNormalization($data, $format);
    }
}
```


