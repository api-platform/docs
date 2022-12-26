# Процессоры состояний

Для изменения состояний приложения во время `POST`, `PUT`, `PATCH` или `DELETE` [операций](operations.md), API Platform использует
классы, называемые **процессорами состояний**. Процессоры состояния получают экземпляр класса, помеченный как ресурс API (обычно с использованием
атрибута `#[ApiResource]`). Этот экземпляр содержит данные, предоставленные клиентом во время [процесса десериализации
](serialization-ru.md).

Процессор состояния, использующий [Doctrine ORM](https://www.doctrine-project.org/projects/orm.html ) входит в состав библиотеки и
включена по умолчанию. Он способен сохранять и удалять объекты, которые также отображаются как [Doctrine entities](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/basic-mapping.html).
Так же доступен процессор состояния [Doctrine MongoDB ODM](https://www.doctrine-project.org/projects/mongodb-odm.html) и может быть включен, следуя [документации MongoDB](mongodb.md ).

Тем не менее, вы можете захотеть:

* хранить данные на других persistence layers (уровнях сохраняемости) (Elasticsearch, внешние веб-службы...)
* не предоставлять публично внутреннюю модель, сопоставленную с базой данных через API
* используйте отдельную модель для [операций чтения](state-providers.md) и для обновлений путем реализации шаблонов, таких как [CQRS](https://martinfowler.com/bliki/CQRS.html )

Для этого можно использовать пользовательские процессоры состояний. Проект может включать в себя столько процессоров состояний, сколько необходимо.
Будет использоваться первый способный обрабатывать данные ресурса.

## Создание пользовательского обработчика состояний

Если [Symfony MakerBundle](https://symfony.com/doc/current/bundles/SymfonyMakerBundle) установлен в вашем проекте, вы можете использовать следующую команду для простого создания пользовательского процессора состояний:

```console
bin/console make:state-processor
```

Чтобы создать обработчик состояний, вы должны реализовать [`ProcessorInterface`](https://github.com/api-platform/core/blob/main/src/State/ProcessorInterface.php).
Этот интерфейс определяет метод `process`: для создания, удаления, обновления данных любыми способами.

Вот пример реализации:

```php
<?php

namespace App\State;

use App\Entity\BlogPost;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;

class BlogPostProcessor implements ProcessorInterface
{
    /**
     * {@inheritDoc}
     */
    public function process($data, Operation $operation, array $uriVariables = [], array $context = [])
    {
        // call your persistence layer to save $data
        return $data;
    }
}
```

Затем мы настраиваем операции для использование этого процессора:

```php
<?php

namespace App\Entity;

use ApiPlatform\Metadata\Post;
use App\State\BlogPostProcessor;

#[Post(processor: BlogPostProcessor::class)]
class BlogPost {}
```

Если служба автозапуска и автоконфигурации включена (они включены по умолчанию), то все готово!

В противном случае, если вы используете пользовательскую конфигурацию внедрения зависимостей, вам необходимо зарегистрировать соответствующую службу и добавить
тег `api_platform.state_processor`.

```yaml
# api/config/services.yaml
services:
    # ...
    App\State\BlogPostProcessor: ~
        # Uncomment only if autoconfiguration is disabled
        #tags: [ 'api_platform.state_processor' ]
```

## Подключение к встроенным процессорам состояния

If you want to execute custom business logic before or after persistence, this can be achieved by [decorating](https://symfony.com/doc/current/service_container/service_decoration.html) the built-in state processors or using [composition](https://en.wikipedia.org/wiki/Object_composition).

The next example uses [Symfony Mailer](https://symfony.com/doc/current/mailer.html). Read its documentation if you want to use it.

Here is an implementation example which sends new users a welcome email after a REST `POST` or GraphQL `create` operation, in a project using the native Doctrine ORM state processor:

Если вы хотите выполнить пользовательскую бизнес-логику до или после сохранения, это может быть достигнуто с помощью [декорирования](https://symfony.com/doc/current/service_container/service_decoration.html)
встроенные процессоры состояния или с использованием [композиции](https://en.wikipedia.org/wiki/Object_composition).

В следующем примере используется [Symfony Mailer](https://symfony.com/doc/current/mailer.html ).
Прочтите его документацию, если вы хотите его использовать.

Вот пример реализации, который отправляет новым пользователям приветственное электронное письмо
после REST операции `POST` или GraphQL `create`, в проекте использующем собственный процессор состояния Doctrine ORM:

```php
<?php

namespace App\State;

use ApiPlatform\Metadata\DeleteOperationInterface;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;
use App\Entity\User;
use Symfony\Component\Mailer\MailerInterface;

final class UserProcessor implements ProcessorInterface
{
    public function __construct(private ProcessorInterface $persistProcessor, private ProcessorInterface $removeProcessor, MailerInterface $mailer)
    {
    }

    public function process($data, Operation $operation, array $uriVariables = [], array $context = [])
    {
        if ($operation instanceof DeleteOperationInterface) {
            return $this->removeProcessor->process($data, $operation, $uriVariables, $context);
        }
    
        $result = $this->persistProcessor->process($data, $operation, $uriVariables, $context);
        $this->sendWelcomeEmail($data);
        return $result;
    }

    private function sendWelcomeEmail(User $user)
    {
        // Your welcome email logic...
        // $this->mailer->send(...);
    }
}
```

Необходимо прописать конфигурацию, даже при включенном автозапуске службы и автоконфигурации:

```yaml
# api/config/services.yaml
services:
    # ...
    App\State\UserProcessor:
        bind:
            $persistProcessor: '@api_platform.doctrine.orm.state.persist_processor'
            $removeProcessor: '@api_platform.doctrine.orm.state.remove_processor'
        # Uncomment only if autoconfiguration is disabled
        #arguments: ['@App\State\UserProcessor.inner']
        #tags: [ 'api_platform.state_processor' ]
```

И настройте, что вы хотите использовать этот процессор на ресурсе User:

```php
<?php

namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use App\State\UserProcessor;

#[ApiResource(processor: UserProcessor::class)]
class User {}
```
