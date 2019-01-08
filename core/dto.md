# Using Data Transfer Objects (DTOs)

## Specifying an Input or an Output Class

For a given resource class, you may want to have a different representation of this class as input (write) or output (read).
To do so, a resource can take an input and/or an output class:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use App\Dto\BookInput;
use App\Dto\BookOutput;

/**
 * @ApiResource(
 *   inputClass=BookInput::class,
 *   outputClass=BookOutput::class
 * )
 */
final class Book
{
}
```

The `input_class` attribute is used during [the deserialization process](serialization.md), when transforming the user provided data to a resource instance.
Similarly, the `output_class` attribute is used during the serialization process, this class represents how the `Book` resource will be represented in the `Response`.

To create a `Book`, we `POST` a data structure corresponding to the `BookInput` class and get back in the response a data structure corresponding to the `BookOuput` class.

To persist the input object, a custom [data persister](data-persisters.md) handling `BookInput` instances must be written.
To retrieve an instance of the output class, a custom [data provider](data-providers.md) returning a `BookOutput` instance must be written.

The `input_class` and `output_class` attributes are taken into account by all the documentation generators (GraphQL and OpenAPI, Hydra).

## Disabling the Input or the Output

Both the `input_class` and the `output_class` attributes can be the to `false`.
If `input_class` is `false`, the deserialization process will be skipped, and the no data persisters will be called.
If `output_class` is `false`, the serialization process will be skipped, and no data providers will be called. 

## Creating a Service-Oriented endpoint

Sometimes it's convenient to create [RPC](https://en.wikipedia.org/wiki/Remote_procedure_call)-like endpoints.
For example, the application should be able to send an email when someone has lost its password.

So let's create a basic DTO for this request:

```php
<?php
// api/src/Api/Dto/ForgotPasswordRequest.php

namespace App\Entity\Dto;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * @ApiResource(
 *     collectionOperations={
 *         "post"={
 *             "path"="/users/forgot-password-request",
 *             "status"=202
 *        },
 *     },
 *     itemOperations={},
 *     outputClass=false
 * )
 */
final class ForgotPasswordRequest
{
    /**
     * @Assert\NotBlank
     * @Assert\Email
     */
    public $email;
}
```

In this case, we disable all operations except `POST`. We also set the `output_class` attribute to `false` to hint
API Platform that no data will be returned by this endpoint.
Finally, we use the `status` attribute to configure API Platform to return a [202 Accepted HTTP status code](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/202).
It indicates that the request has been received and will be treated without giving an immediate return to the client.

Then, thanks to [a custom data persister](data-persisters.md), it's possible to trigger some custom logic when the request is received.

Create the data persister:

```php
// api/src/DataPersister/ForgotPasswordRequestDataPersister.php
namespace App\DataPersister;

use ApiPlatform\Core\DataPersister\DataPersisterInterface;
use App\Entity\ForgotPasswordRequest;

final class ForgotPasswordRequestDataPersister implements DataPersisterInterface
{
    public function supports($data): bool
    {
        return $data instanceof ForgotPasswordRequest;
    }
    
    public function persist($data)
    {
      // Trigger your custom logic here
      return $data;
    }
    
    public function remove($data)
    {
      throw new \RuntimeException('"remove" is not supported');
    }
}
```

And register it:

```yaml
# api/config/services.yaml
services:
    # ...
    'App\DataPersister\ForgotPasswordRequestDataPersister': ~
        # Uncomment only if autoconfiguration is disabled
        #tags: [ 'api_platform.data_persister' ]
```

Instead of a custom data provider, you'll probably want to leverage the Symfony Messenger Component integration.
