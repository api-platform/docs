# Using Data Transfer Objects (DTOs)

As stated in [the general design considerations](design.md), in most cases [the DTO pattern](https://en.wikipedia.org/wiki/Data_transfer_object) should be implemented using an API Resource class representing the public data model exposed through the API and [a custom State Provider](state-providers.md). In such cases, the class marked with `#[ApiResource]` will act as a DTO.

However, it's sometimes useful to use a specific class to represent the input or output data structure related to an operation. These techniques are very useful to document your API properly (using Hydra or OpenAPI) and will often be used on `POST` operations.

## Implementing a Write Operation With an Input Different From the Resource

Using an input, the request body will be denormalized to the input instead of your resource class.

```php
<?php
// api/src/Dto/UserResetPasswordDto.php
namespace App\Dto;

use Symfony\Component\Validator\Constraints as Assert;

final class UserResetPasswordDto
{
    #[Assert\Email]
    public $email;
}
```

```php
<?php
// api/src/Model/User.php
namespace App\Model;

use ApiPlatform\Metadata\Post;
use App\Dto\UserResetPasswordDto;
use App\State\UserResetPasswordProcessor;

#[Post(input: UserResetPasswordDto::class, processor: UserResetPasswordProcessor::class)]
final class User {}
```

And the processor:

```php
<?php

namespace App\State;

use App\Dto\UserResetPasswordDto;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

final class UserResetPasswordProcessor implements ProcessorInterface
{
    /**
     * @param UserResetPasswordDto $data
     */
    public function process($data, Operation $operation, array $uriVariables = [], array $context = [])
    {
        if ('user@example.com' === $data->email) {
            return new User(email: $data->email, id: 1);
        }

        throw new NotFoundHttpException();
    }
}
```

In some cases, using an input DTO is a way to avoid serialization groups.

## Use Messenger With an Input DTO

Let's use a message that will be processed by [Symfony Messenger](https://symfony.com/components/Messenger). API Platform has an [integration with messenger](./messenger.md), to use a DTO as input you need to specify the `input` attribute:

```php
<?php

namespace App\Model;

use ApiPlatform\Metadata\Post;
use ApiPlatform\Symfony\Messenger\Processor as MessengerProcessor;
use App\Dto\Message;

#[Post(input: Message::class, processor: MessengerProcessor::class)]
class SendMessage {}
```

This will dispatch the `App\Dto\Message` via [Symfony Messenger](https://symfony.com/components/Messenger).

## Implementing a Read Operation With an Output Different From the Resource

To return another representation of your data in a [State Provider](./state-providers.md) we advise to specify the `output` attribute of the resource. Note that this technique works without any changes to the resource but your API documentation would be wrong.

```php
<?php

namespace App\Entity;

use ApiPlatform\Metadata\Get;
use App\Dto\AnotherRepresentation;
use App\State\BookRepresentationProvider;

#[Get(output: AnotherRepresentation::class, provider: BookRepresentationProvider::class)]
class Book {}
```

```php
<?php

namespace App\State;

use App\Dto\AnotherRepresentation;
use App\Model\Book;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;

final class BookRepresentationProvider implements ProviderInterface
{
    public function provide(Operation $operation, array $uriVariables = [], array $context = [])
    {
        return new AnotherRepresentation();
    }
}
```

## Implementing a Write Operation With an Output Different From the Resource

For returning another representation of your data in a [State Processor](./state-processors.md), you should specify your processor class in the `processor` attribute and same for your `output`.

<code-selector>

```php
<?php

namespace App\Entity;

use ApiPlatform\Metadata\Post;
use App\Dto\AnotherRepresentation;
use App\State\BookRepresentationProcessor;

#[Post(output: AnotherRepresentation::class, processor: BookRepresentationProcessor::class)]
class Book {}
```
```yaml
# api/config/api_platform/resources.yaml
App\Entity\Book:
    operations:
        ApiPlatform\Metadata\Post:
            output: App\Dto\AnotherRepresentation
            processor: App\State\BookRepresentationProcessor
```
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
        https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <operations>
            <operation class="ApiPlatform\Metadata\Post" 
                       processor="App\State\BookRepresentationProcessor"
                       output="App\Dto\AnotherRepresentation" /> 
        </operations>
    </resource>
</resources>
```

</code-selector>

Here the `$data` attribute represents an instance of your resource.

```php
<?php

namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;
use App\Dto\AnotherRepresentation;
use App\Model\Book;

final class BookRepresentationProcessor implements ProcessorInterface
{
     /**
     * @param Book $data
     */
    public function process($data, Operation $operation, array $uriVariables = [], array $context = [])
    {
        return new AnotherRepresentation(
            $data->getId(),
            $data->getTitle(),
            // etc.
        );
    }
}
```
