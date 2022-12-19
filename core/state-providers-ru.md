# Поставщики Состояний

Для извлечения данных, предоставляемых API, API Platform использует классы, называемые **поставщиками состояний**. 
В библиотеку сключены поставщики состояний, использующий [Doctrine ORM](https://www.doctrine-project.org/projects/orm.html) для извлечения данных из базы данных, 
[Doctrine MongoDB ODM](https://www.doctrine-project.org/projects/mongodb-odm.html) для извлечения данных из
базы данных документов и 
[Elasticsearch-PHP](https://www.elastic.co/guide/en/elasticsearch/client/php-api/current/index.html)
для извлечения данных из кластера Elasticsearch. Первый из них включен по умолчанию.
Эти поставщики состояний изначально поддерживают коллекции и фильтры с выгрузкой по страницам.
Они могут использоваться как есть и идеально подходят для обычных целей.

Однако иногда требуется извлечь данные из других источников, таких как другой уровень сохраняемости(persistence layer) или веб-сервис.
Для этого можно использовать поставщиков пользовательских состояний. 
Проект может включать столько поставщиков, сколько необходимо. Будет использоваться первый способ получения данных для ресурса.

Для этого вам необходимо реализовать `ApiPlatform\State\ProviderInterface`.

In the following examples we will create custom state providers for an entity class called `App\Entity\BlogPost`.
Note, that if your entity is not Doctrine-related, you need to flag the identifier property by using
`#[ApiProperty(identifier: true)` for things to work properly (see also [Entity Identifier Case](serialization.md#entity-identifier-case)).

В следующих примерах мы создадим пользовательские поставщики состояний для класса сущностей с именем `App\Entity\BlogPost`.
Обратите внимание, что если ваша сущность не связана с Doctrine, вам необходимо отметить свойство identifier с помощью
`#[ApiProperty(identifier: true)` для правильной работы (см. также [Случай идентификатора сущности](serialization-ru.md#случай-идентификатора-сущности-entity-identifier-case)).

## Создание пользовательского поставщика состояний

Если в проекте установлен [Symfony MakerBundle](https://symfony.com/doc/current/bundles/SymfonyMakerBundle),
вы можете создать поставщика при помощи следующей команды:

```console
bin/console make:state-provider
```

Давайте начнем с поставщика для URI: `/blog_posts/{id}`.

Ваш `BlogPostProvider` должен реализовать
[`ProviderInterface`](https://github.com/api-platform/core/blob/main/src/State/ProviderInterface.php):

```php
<?php

namespace App\State;

use App\Entity\BlogPost;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;

final class BlogPostProvider implements ProviderInterface
{
    /**
     * {@inheritDoc}
     */
    public function provide(Operation $operation, array $uriVariables = [], array $context = [])
    {
        return new BlogPost($uriVariables['id']);
    }
}
```

As this operation expects a BlogPost we return an instance of the BlogPost in the `provide` method.
The `uriVariables` parameter is an array with the values of the URI variables.

To use this provider we need to configure the provider on the operation:

Поскольку эта операция ожидает BlogPost, мы возвращаем экземпляр BlogPost в методе `provide`.
Параметр `urlvariables` представляет собой массив со значениями переменных URI.

Чтобы использовать этот провайдера, нам нужно настроить операцию с ним:

```php
<?php

namespace App\Entity;

use ApiPlatform\Metadata\Get;
use App\State\BlogPostProvider;

#[Get(provider: BlogPostProvider::class)]
class BlogPost {}
```

Если вы используете конфигурацию по умолчанию, соответствующая сервис будет автоматически зарегистрирована благодаря
[autowiring](https://symfony.com/doc/current/service_container/autowiring.html).
Чтобы явно объявить сервис, вы можете использовать следующий фрагмент кода:

```yaml
# api/config/services.yaml
services:
    # ...
    App\State\BlogPostProvider: ~
        # Раскомментируйте только в том случае, если автоконфигурация отключена
        #tags: [ 'api_platform.state_provider' ]
```

Теперь давайте предположим, что мы также хотим обработать URL-адрес `/blog_posts`, который возвращает коллекцию.
Мы можем изменить это в поставщике, чтобы он поддерживал более широкий спектр операций.
Затем мы можем предоставить коллекцию записей в блоге, когда операция является "CollectionOperationInterface":

```php
<?php

namespace App\State;

use App\Entity\BlogPost;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;
use ApiPlatform\Metadata\CollectionOperationInterface;

final class BlogPostProvider implements ProviderInterface
{
    /**
     * {@inheritDoc}
     */
    public function provide(Operation $operation, array $uriVariables = [], array $context = [])
    {
        if ($operation instanceof CollectionOperationInterface) {
            return [new BlogPost(), new BlogPost()];
        }

        return new BlogPost($uriVariables['id']);
    }
}
```

Затем нам нужно настроить этого же поставщика для операции `getCollection` в BlogPost, или для каждой операции с помощью атрибута `ApiResource`:
```php
<?php

namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use App\State\BlogPostProvider;

#[ApiResource(provider: BlogPostProvider::class)]
class BlogPost {}
```

## Подключение к встроенному поставщику состояния

Если вы хотите выполнить пользовательскую бизнес-логику до или после извлечения данных, это может быть достигнуто с помощью [декорирования](https://symfony.com/doc/current/service_container/service_decoration.html)
встроенных поставщиков состояний или с использованием [композиции](https://en.wikipedia.org/wiki/Object_composition ).

В следующем примере используется [DTO](https://api-platform.com/docs/core/dto/#using-data-transfer-objects-dtos),
чтобы изменить представление данных, первоначально полученных поставщиком состояния по умолчанию.

```php
<?php

namespace App\State;

use App\Dto\AnotherRepresentation;
use App\Model\Book;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;

final class BookRepresentationProvider implements ProviderInterface
{
    public function __construct(private ProviderInterface $itemProvider)
    {
    }
    
    public function provide(Operation $operation, array $uriVariables = [], array $context = []): object|array|null
    {
        $book = $this->itemProvider->provide($operation, $uriVariables, $context);
        
        return new AnotherRepresentation(
            // Добавьте сюда параметры конструктора DTO.
            // $book->getTitle(),
        );
    }
}
```


Даже при включенном autowiring и автоконфигурации, вы все равно должны настроить конфигурацию:

```yaml
# api/config/services.yaml
services:
    # ...
    App\State\BookRepresentationProvider:
        bind:
            $itemProvider: '@api_platform.doctrine.orm.state.item_provider'
        # Раскомментируйте только в том случае, если автоконфигурация отключена
        #tags: [ 'api_platform.state_provider' ]
```

И настройте, что вы хотите использовать этого поставщика в ресурсе Book:

```php
<?php

namespace App\Entity;

use ApiPlatform\Metadata\Get;
use App\Dto\AnotherRepresentation;
use App\State\BookRepresentationProvider;

#[Get(output: AnotherRepresentation::class, provider: BookRepresentationProvider::class)]
class Book {}
```
