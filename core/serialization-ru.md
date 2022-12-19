# Cериализация

## Объяснение процесса

API Platform включает и расширяет компонент Symfony Serializer для преобразования объектов(сущностей) PHP в ответах API (гипермедиа).


<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/serializer?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Serializer screencast"><br>Посмотрите скринкаст про Serializer</a></p>

Основной процесс сериализации состоит из двух этапов:

![Рабочий процесс сериализатора](images/SerializerWorkflow.png)

> Как вы можете видеть на рисунке выше, массив используется как посредник. Таким образом, кодировщики(Encoders) будут иметь дело только с преобразованием определенных форматов в массивы и наоборот. Таким же образом нормализаторы(Normalizers) будут иметь дело с превращением определенных объектов в массивы и наоборот.

-- [Документация Symfony](https://symfony.com/doc/current/components/serializer.html)

В отличие от Symfony, API Platform использует пользовательские нормализаторы, свой роутер и систему для выполнения расширенного преобразования [state provider](state-providers.md). К сгенерированному документу добавляются метаданные, включая ссылки, информацию о типе, данные о разбивке на страницы или доступные фильтры.

Сериализатор API Platform можно расширять. Вы можете зарегистрировать пользовательские нормализаторы (Normalizers) и кодировщики (Encoders) для поддержки других форматов. Вы также можете настроить поведение существующих нормализаторов, обернув их в свой [декоратор](https://refactoring.guru/ru/design-patterns/decorator).


## Доступные сериализаторы

* Сериализатор [JSON-LD](https://json-ld.org)
`api_platform.jsonld.normalizer.item`

JSON-LD, или объектная нотация JavaScript для связанных данных - это метод кодирования связанных данных с использованием JSON. Это рекомендация  [W3C](https://www.w3.org/TR/json-ld11/).

* Сериализатор [HAL](https://en.wikipedia.org/wiki/Hypertext_Application_Language)
`api_platform.hal.normalizer.item`

* Сериализаторы JSON, XML, CSV, YAML (использование сериализатора Symfony)
`api_platform.serializer.normalizer.item`

## Контекст сериализации, группы и отношения

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/serialization-groups?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Serialization Groups screencast"><br>Смотрите скринкаст Serialization Groups</a></p>

API Platform позволяет вам указать переменную `$context`, используемую сериализатором Symfony. Эта переменная представляет собой ассоциативный массив, который имеет удобный ключ "группы", позволяющий вам выбирать, какие атрибуты ресурса отображаются во время процессов нормализации (чтения) и денормализации (записи).
Он опирается на [группы сериализации (и десериализации)](https://symfony.com/doc/current/components/serializer.html#attributes-groups)
особенность компонента Symfony Serializer.

В дополнение к группам вы можете использовать любую опцию, поддерживаемую сериализатором Symfony. Например, вы можете использовать [`enable_max_depth`](https://symfony.com/doc/current/components/serializer.html#handling-serialization-depth)
чтобы ограничить глубину сериализации.

### Конфигурация

Как и другие компоненты Symfony и API Platform, компонент Serializer можно настроить с помощью аннотаций, XML или YAML. Поскольку аннотации легки для понимания, мы будем использовать их в следующих примерах.

Примечание: если вы не используете дистрибутив API Platform, вам нужно будет включить поддержку аннотаций в конфигурации сериализатора:

```yaml
# api/config/packages/framework.yaml
framework:
    serializer: { enable_annotations: true }
```

Если вы используете [Symfony Flex](https://github.com/symfony/flex), просто выполните
`composer req doctrine/annotations`
и всё готово!

Если вы хотите использовать YAML или XML, пожалуйста, добавьте путь сопоставления в конфигурацию сериализатора:

```yaml
# api/config/packages/framework.yaml
framework:
    serializer:
        mapping:
            paths: ['%kernel.project_dir%/config/serialization']
```

## Использование групп сериализации

Просто указать, какие группы использовать в системе API:

1. Добавьте атрибуты контекста нормализации и контекста денормализации к ресурсу и укажите, какие группы использовать. Здесь вы видите, что мы добавляем `read` и `write` соответственно. Вы можете использовать любые названия групп, которые пожелаете.
2. Примените группы к свойствам объекта.

[codeSelector]

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

#[ApiResource(
    normalizationContext: ['groups' => ['read']],
    denormalizationContext: ['groups' => ['write']],
)]
class Book
{
    #[Groups(['read', 'write'])]
    public $name;

    #[Groups('write')]
    public $author;

    // ...
}
```

```yaml
# api/config/api_platform/resources.yaml
resources:
    App\Entity\Book:
        normalizationContext:
            groups: ['read']
        denormalizationContext:
            groups: ['write']

# api/config/serialization/Book.yaml
App\Entity\Book:
    attributes:
        name:
            groups: ['read', 'write']
        author:
            groups: ['write']
```

```xml
<!-- api/config/api_platform/resources.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
                               https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <normalizationContext>
            <values>
                <value name="groups">
                    <values>
                        <value>read</value>
                    </values>
                </value>
            </values>
        </normalizationContext>
        <denormalizationContext>
            <values>
                <value name="groups">
                    <values>
                        <value>write</value>
                    </values>
                </value>
            </values>
        </denormalizationContext>
    </resource>
</resources>

<!-- api/config/serialization/Book.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<serializer xmlns="http://symfony.com/schema/dic/serializer-mapping"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://symfony.com/schema/dic/serializer-mapping
                                http://symfony.com/schema/dic/serializer-mapping/serializer-mapping-1.0.xsd">
    <class name="App\Entity\Book">
        <attribute name="name">
            <group>read</group>
            <group>write</group>
        </attribute>
        <attribute name="author">
            <group>write</group>
        </attribute>
    </class>
</serializer>
```

[/codeSelector]

В предыдущем примере свойство `name` будет видно при чтении (`GET`) объекта, и оно также будет доступно
для записи (`PUT` / `PATCH` / `POST`). Свойство `author` будет доступно только для записи; оно не будет видно, когда
API возвращает сериализованные ответы.

Внутри API Platform передает значение `normalizationContext` в качестве третьего аргумента [метода `Serializer::serialize()`](https://api.symfony.com/master/Symfony/Component/Serializer/SerializerInterface.html#method_serialize) во время процесса нормализации.
 `denormalizationContext` передается в качестве 4-го аргумента [метода `Serializer::deserialize()`](https://api.symfony.com/master/Symfony/Component/Serializer/SerializerInterface.html#method_deserialize) во время денормализации ( записи).


To configure the serialization groups of classes's properties, you must use directly [the Symfony Serializer's configuration files or annotations](https://symfony.com/doc/current/components/serializer.html#attributes-groups).

In addition to the `groups` key, you can configure any Symfony Serializer option through the `$context` parameter
(e.g. the `enable_max_depth`key when using [the `@MaxDepth` annotation](https://symfony.com/doc/current/components/serializer.html#handling-serialization-depth)).

Any serialization and deserialization group that you specify will also be leveraged by the built-in actions and the Hydra
documentation generator.

Чтобы настроить группы сериализации свойств классов, вы должны использовать напрямую
[файлы конфигурации Symfony Serializer или аннотации](https://symfony.com/doc/current/components/serializer.html#attributes-groups).

В дополнение к ключу `groups`, вы можете настроить любую опцию Symfony Serializer через параметр `$context`
(например, ключ `enable_max_depth` при использовании [аннотации `@MaxDepth`](https://symfony.com/doc/current/components/serializer.html#handling-serialization-depth)).

Любая указанная вами группа сериализации и десериализации также будет использоваться встроенными действиями(actions) и генератором документации Hydra.

## Использование групп сериализации в операцииях

Можно указать контексты нормализации и денормализации (а также любой другой атрибут) для каждой операции.
API Platform всегда будет использовать наиболее конкретное определение. Например, если группы нормализации установлены как на уровне ресурсов, так и на уровне операций, то будет использоваться конфигурация, установленная на уровне операций, а уровень ресурсов игнорируется.

В следующем примере мы используем разные группы сериализации для операций `GET` и `PUT`:

[codeSelector]

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\Put;
use Symfony\Component\Serializer\Annotation\Groups;

#[ApiResource(normalizationContext: ['groups' => ['get']])]
#[Get]
#[Put(normalizationContext: ['groups' => ['put']])]
class Book
{
    #[Groups(['get', 'put'])
    public $name;

    #[Groups('get')]
    public $author;

    // ...
}
```

```yaml
# api/config/api_platform/resources/Book.yaml
App\Entity\Book:
    normalizationContext:
        groups: ['get']
    operations:
        ApiPlatform\Metadata\Get: ~
        ApiPlatform\Metadata\Put:
            normalizationContext:
                groups: ['put']

# api/config/serializer/Book.yaml
App\Entity\Book:
    attributes:
        name:
            groups: ['get', 'put']
        author:
            groups: ['get']
```

```xml
<!-- api/config/api_platform/resources.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
                               https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <normalizationContext>
            <values>
                <value name="groups">
                    <values>
                        <value>get</value>
                    </values>
                </value>
            </values>
        </normalizationContext>
        <operations>
            <operation class="ApiPlatform\Metadata\Get" />
            <operation class="ApiPlatform\Metadata\Put">
                <normalizationContext>
            <values>
                <value name="groups">
                    <values>
                        <value>put</value>
                    </values>
                </value>
            </values>
        </normalizationContext>
            </operation>
        </operations>
    </resource>
</resources>

<!-- api/config/serialization/Book.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<serializer xmlns="http://symfony.com/schema/dic/serializer-mapping"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://symfony.com/schema/dic/serializer-mapping
                                http://symfony.com/schema/dic/serializer-mapping/serializer-mapping-1.0.xsd">
    <class name="App\Entity\Book">
        <attribute name="name">
            <group>get</group>
            <group>put</group>
        </attribute>
        <attribute name="author">
            <group>get</group>
        </attribute>
    </class>
</serializer>
```

[/codeSelector]

The `name` and `author` properties will be included in the document generated during a `GET` operation because the configuration
defined at the resource level is inherited. However the document generated when a `PUT` request will be received will only
include the `name` property because of the specific configuration for this operation.

Refer to the [operations](operations.md) documentation to learn more.
Свойства `name` и `author` будут включены в документ, сгенерированный во время операции `GET`, поскольку наследуется конфигурация определенная на уровне ресурсов.
Однако документ, сгенерированный при получении запроса `PUT`, будет
содержать только свойство `name` из-за конфигурации указанной для этой операции.

Обратитесь к [операциям](operations.md ) документация, чтобы узнать больше.

## Встраивание отношений (Embedding Relations)

По умолчанию сериализатор, предоставляемый API Platform, представляет отношения между объектами, используя [разыменовываемый IRIs](https://en.wikipedia.org/wiki/Internationalized_Resource_Identifier ).
Они позволяют вам извлекать сведения о связанных объектах путем выдачи дополнительных HTTP-запросов. Однако по соображениям производительности иногда предпочтительнее не заставлять клиента производить дополнительные HTTP-запросы.

**Примечание:** Мы настоятельно рекомендуем использовать [Vulcain](https://vulcain.rocks) вместо этой функции. Vulcain позволяет создавать более быстрые (с большей частотой посещений) и лучше спроектированные API, чем полагаться на составные документы, и поддерживается из коробки в дистрибутиве API Platform.

### Нормализация

В следующем документе JSON отношение книги к автору по умолчанию представлено URI:

```json
{
  "@context": "/contexts/Book",
  "@id": "/books/62",
  "@type": "Book",
  "name": "My awesome book",
  "author": "/people/59"
}
```

Можно встроить связанные объекты (полностью или только некоторые из их свойств) непосредственно в родительский
ответ с помощью групп сериализации. Используя следующие аннотации групп сериализации (`#[Groups]`),
представление автора в формате JSON встроено в ответ на запрос книги. Как только какой-либо из атрибутов автора окажется в группе `book`, автор будет внедрен в этот ответ.

[codeSelector]

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

#[ApiResource(normalizationContext: ['groups' => ['book']])]
class Book
{
    #[Groups('book')]
    public $name;

    #[Groups('book')]
    public $author;

    // ...
}
```

```yaml
# api/config/api_platform/resources/Book.yaml
App\Entity\Book:
    normalizationContext:
        groups: ['book']

# api/config/serializer/Book.yaml
App\Entity\Book:
    attributes:
        name:
            groups: ['book']
        author:
            groups: ['book']
```

[/codeSelector]

[codeSelector]

```php
<?php
// api/src/Entity/Person.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

#[ApiResource]
class Person
{
    #[Groups('book')]
    public $name;

    // ...
}
```

```yaml
# api/config/serializer/Person.yaml
App\Entity\Person:
    attributes:
        name:
            groups: ['book']
```

[/codeSelector]

Сгенерированный JSON с использованием предыдущих настроек приведен ниже:

```json
{
  "@context": "/contexts/Book",
  "@id": "/books/62",
  "@type": "Book",
  "name": "My awesome book",
  "author": {
    "@id": "/people/59",
    "@type": "Person",
    "name": "Kévin Dunglas"
  }
}
```

Чтобы оптимизировать такие встроенные отношения, поставщик состояний Doctrine (Doctrine state provider) по умолчанию автоматически присоединяет сущности к отношениям  отмечен как [`EAGER`](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/annotations-reference.html#manytoone).
Это позволяет избежать выполнения дополнительных запросов при сериализации связанных объектов.

Вместо того, чтобы встраивать отношения в основной HTTP-ответ, вы можете захотеть ["отправить" их клиенту, используя HTTP/2 server push](push-relations.md).

### Денормализация

Также возможно встроить отношение в запросы `PUT`, `PATCH` и `POST`. Чтобы включить эту функцию, установите группы сериализации
таким же образом, как и нормализацию. Например:

[codeSelector]

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;

#[ApiResource(denormalizationContext: ['groups' => ['book']])]
class Book
{
    // ...
}
```

```yaml
# api/config/api_platform/resources/Book.yaml
App\Entity\Book:
    denormalizationContext:
        groups: ['book']
```

[/codeSelector]


При денормализации встроенных отношений применяются следующие правила:

* Если ключ `@id` присутствует во встроенном ресурсе, то объект, соответствующий данному URI, будет извлечен через поставщика состояния(state provider). Любые изменения во встроенном отношении также будут применены к этому объекту.
* Если ключ `@id` не существует, будет создан новый объект, содержащий состояние, указанное во встроенном документе JSON.

Вы можете указать столько встроенных уровней отношений, сколько захотите.


### Принудительный IRI с отношениями того же типа (отношения родитель/потомок)

Распространенной проблемой является наличие объектов, которые ссылаются на другие объекты того же типа:

[codeSelector]

```php
<?php
// api/src/Entity/Person.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

#[ApiResource(
    normalizationContext: ['groups' => ['person']],
    denormalizationContext: ['groups' => ['person']]
)]
class Person
{
    #[Groups('person')]
    public $name;

   /**
    * @var Person
    */
    #[Groups('person')]
   public $parent;  // Обратите внимание, что экземпляр Person связан с другим Person.
 
    // ...
}
```

```yaml
# api/config/api_platform/resources/Person.yaml
App\Entity\Person:
    normalizationContext:
        groups: ['person']
    denormalizationContext:
        groups: ['person']

# api/config/serializer/Person.yaml
App\Entity\Person:
    attributes:
        name:
            groups: ['person']
        parent:
            groups: ['person']
```

[/codeSelector]

Проблема здесь в том, что свойство **$parent** автоматически становится встроенным объектом. Кроме того, свойство не будет отображаться в представлении OpenAPI.

Чтобы принудительно использовать свойство **$parent** в качестве IRI, добавьте аннотацию `#[ApiProperty(readableLink: false, writableLink: false)]`:

[codeSelector]

```php
<?php
// api/src/Entity/Person.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

#[ApiResource(
    normalizationContext: ['groups' => ['person']],
    denormalizationContext: ['groups' => ['person']]
)]
class Person
{
    #[Groups('person')]
    public string $name;

   #[Groups('person')]
   #[ApiProperty(readableLink: false, writableLink: false)]
   public Person $parent;  // Это свойство теперь сериализуется/десериализуется как IRI.
 
    // ...
}

```

```yaml
# api/config/api_platform/resources/Person.yaml
resources:
    App\Entity\Person:
        normalizationContext:
          groups: ['person']
        denormalizationContext:
          groups: ['person']
properties:
    App\Entity\Person:
        parent:
            readableLink: false
            writableLink: false

# api/config/serializer/Person.yaml
App\Entity\Person:
    attributes:
        name:
            groups: ['person']
        parent:
            groups: ['person']
```

[/codeSelector]

### Простые идентификаторы

Вместо отправки IRI для установки отношения вы можете отправить простой идентификатор. Для этого вы должны создать свой собственный денормализатор:

```php
<?php
// api/src/Serializer/PlainIdentifierDenormalizer

namespace App\Serializer;

use ApiPlatform\Api\IriConverterInterface;
use App\Entity\Dummy;
use App\Entity\RelatedDummy;
use Symfony\Component\Serializer\Normalizer\ContextAwareDenormalizerInterface;
use Symfony\Component\Serializer\Normalizer\DenormalizerAwareInterface;
use Symfony\Component\Serializer\Normalizer\DenormalizerAwareTrait;

class PlainIdentifierDenormalizer implements ContextAwareDenormalizerInterface, DenormalizerAwareInterface
{
    use DenormalizerAwareTrait;

    private $iriConverter;

    public function __construct(IriConverterInterface $iriConverter)
    {
        $this->iriConverter = $iriConverter;
    }

    /**
     * {@inheritdoc}
     */
    public function denormalize($data, $class, $format = null, array $context = [])
    {
        $data['relatedDummy'] = $this->iriConverter->getIriFromResource(resource: RelatedDummy::class, context: ['uri_variables' => ['id' => $data['relatedDummy']]]);

        return $this->denormalizer->denormalize($data, $class, $format, $context + [__CLASS__ => true]);
    }

    /**
     * {@inheritdoc}
     */
    public function supportsDenormalization($data, $type, $format = null, array $context = []): bool
    {
        return \in_array($format, ['json', 'jsonld'], true) && is_a($type, Dummy::class, true) && !empty($data['relatedDummy']) && !isset($context[__CLASS__]);
    }
}
```

## Контекст нормализации свойств

Если вы хотите изменить контекст (де)нормализации свойства, например, если вы хотите изменить формат даты и времени,
вы можете сделать это, используя атрибут `#[Context]` компонента Symfony Serializer.

Например:

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Serializer\Annotation\Context;
use Symfony\Component\Serializer\Normalizer\DateTimeNormalizer;

#[ORM\Entity]
#[ApiResource]
class Book
{
    #[ORM\Column] 
    #[Context([DateTimeNormalizer::FORMAT_KEY => 'Y-m-d'])]
    public ?\DateTimeInterface $publicationDate = null;
}
```

В приведенном выше примере вы получите данные книги следующим образом:

```json
{
  "@context": "/contexts/Book",
  "@id": "/books/3",
  "@type": "https://schema.org/Book",
  "publicationDate": "1989-06-16"
}
```

Также возможно изменить только контекст денормализации или нормализации:

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Serializer\Annotation\Context;
use Symfony\Component\Serializer\Normalizer\DateTimeNormalizer;

#[ORM\Entity]
#[ApiResource]
class Book
{
    #[ORM\Column]
    #[Context(normalizationContext: [DateTimeNormalizer::FORMAT_KEY => 'Y-m-d'])]
    public ?\DateTimeInterface $publicationDate = null;
}
```

Также поддерживаются группы:

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Serializer\Annotation\Context;
use Symfony\Component\Serializer\Annotation\Groups;
use Symfony\Component\Serializer\Normalizer\DateTimeNormalizer;

#[ORM\Entity]
#[ApiResource]
class Book
{
    #[ORM\Column]
    #[Groups(["extended"])]
    #[Context([DateTimeNormalizer::FORMAT_KEY => \DateTime::RFC3339])]
    #[Context(
        context: [DateTimeNormalizer::FORMAT_KEY => \DateTime::RFC3339_EXTENDED],
        groups: ['extended'],
    )]
    public ?\DateTimeInterface $publicationDate = null;
}
```

## Вычисляемые поля

Иногда вам нужно предоставить вычисляемые поля. Это можно сделать, используя группы. На этот раз не по свойству, а по методу.

[codeSelector]

```php
<?php
// api/src/Entity/Greeting.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Serializer\Annotation\Groups;

#[ApiResource]
#[GetCollection(normalizationContext: ['groups' => 'greeting:collection:get'])]
class Greeting
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    #[Groups("greeting:collection:get")]
    private ?int $id = null;
    
    private $a = 1;
    
    private $b = 2;

    #[ORM\Column]
    #[Groups("greeting:collection:get")]
    public string $name = '';

    public function getId(): int
    {
        return $this->id;
    }

    #[Groups('greeting:collection:get')] // <- МАГИЯ ЗДЕСЬ, вы можете установить группу для метода.
    public function getSum(): int
    {
        return $this->a + $this->b;
    }
}
```

```yaml
# api/config/api_platform/resources/Greeting.yaml
App\Entity\Greeting:
    operations:
        ApiPlatform\Metadata\GetCollection:
            normalizationContext:
                groups: 'greeting:collection:get'

# api/config/serializer/Greeting.yaml
App\Entity\Greeting:
    attributes:
        id:
            groups: 'greeting:collection:get'
        name:
            groups: 'greeting:collection:get'
        sum:
            groups: 'greeting:collection:get'
```

[/codeSelector]

## Динамическое изменение контекста сериализации

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform-security/service-decoration?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Context Builder & Service Decoration screencast"><br>Смотрите скринкаст Context Builder & Service Decoration</a></p>

Давайте представим себе ресурс, где большинством полей может управлять любой пользователь, но некоторыми могут управлять только пользователи-администраторы:

[codeSelector]

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

#[ApiResource(
    normalizationContext: ['groups' => ['book:output']],
    denormalizationContext: ['groups' => ['book:input']],
)]
class Book
{
    // ...

    /**
     * Этим полем может управлять только администратор
     */
    #[Groups(['book:output', 'admin:input'])]
    public bool $active = false;

    /**
     * Этим полем может управлять любой пользователь
     */
    #[Groups(['book:output', 'book:input'])]
    public string $name;

    // ...
}
```

```yaml
# api/config/api_platform/resources/Book.yaml
App\Entity\Book: 
    normalizationContext:
        groups: ['book:output']
    denormalizationContext:
        groups: ['book:input']

# api/config/serializer/Book.yaml
App\Entity\Book:
    attributes:
        active:
            groups: ['book:output', 'admin:input']
        name:
            groups: ['book:output', 'book:input']
```

[/codeSelector]

Все точки входа одинаковы для всех пользователей, поэтому мы должны найти способ определить, является ли аутентифицированный пользователь администратором, и если да, то
динамически добавлять значение `admin:input` в группы десериализации в массиве `$context`.

API Platform реализует "ContextBuilder", который подготавливает контекст для сериализации и десериализации. Давайте
[обернем этот сервис](http://symfony.com/doc/current/service_container/service_decoration.html ), чтобы переопределить метод `createFromRequest`:

```yaml
# api/config/services.yaml
services:
    # ...
    'App\Serializer\BookContextBuilder':
        decorates: 'api_platform.serializer.context_builder'
        arguments: [ '@App\Serializer\BookContextBuilder.inner' ]
        autoconfigure: false
```

```php
<?php
// api/src/Serializer/BookContextBuilder.php
namespace App\Serializer;

use ApiPlatform\Serializer\SerializerContextBuilderInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface;
use App\Entity\Book;

final class BookContextBuilder implements SerializerContextBuilderInterface
{
    private $decorated;
    private $authorizationChecker;

    public function __construct(SerializerContextBuilderInterface $decorated, AuthorizationCheckerInterface $authorizationChecker)
    {
        $this->decorated = $decorated;
        $this->authorizationChecker = $authorizationChecker;
    }

    public function createFromRequest(Request $request, bool $normalization, ?array $extractedAttributes = null): array
    {
        $context = $this->decorated->createFromRequest($request, $normalization, $extractedAttributes);
        $resourceClass = $context['resource_class'] ?? null;

        if ($resourceClass === Book::class && isset($context['groups']) && $this->authorizationChecker->isGranted('ROLE_ADMIN') && false === $normalization) {
            $context['groups'][] = 'admin:input';
        }

        return $context;
    }
}
```

Если у пользователя есть разрешение `ROLE_ADMIN`, а ресурс является экземпляром книги, группа `admin:input` будет динамически добавлена к контекст денормализации.
Переменная `$normalization` позволяет вам проверить, предназначен ли контекст для нормализации (если `TRUE`) или для денормализации (`FALSE`).

## Изменение контекста сериализации для каждого элемента

В приведенном выше примере показано, как вы можете изменить контекст нормализации/денормализации на основе текущих разрешения пользователя для всех книг. Однако иногда разрешения различаются в зависимости от того, какая книга обрабатывается.

Подумайте о ACL: пользователь «А» может получить книгу «А», но не книгу «Б». В этом случае нужно использовать мощь Symfony Serializer и зарегистрировать наш собственный нормализатор, который добавляет группу к каждому отдельному элементу
(примечание: приоритет `64` пример; всегда важно убедиться, что ваш нормализатор загружается первым, поэтому установите приоритет на любое подходящее значение для вашего приложения; более высокие значения загружаются раньше):

```yaml
# api/config/services.yaml
services:
    'App\Serializer\BookAttributeNormalizer':
        arguments: [ '@security.token_storage' ]
        tags:
            - { name: 'serializer.normalizer', priority: 64 }
```

The Normalizer class is a bit harder to understand, because it must ensure that it is only called once and that there is no recursion.
To accomplish this, it needs to be aware of the parent Normalizer instance itself.

Here is an example:

Класс Normalizer получиться немного сложнее, так как он должен гарантировать, что он вызывается только один раз и что рекурсия отсутствует.
Чтобы достичь этого, ему необходимо знать о родительском экземпляре нормализатора.

Вот пример:

```php
<?php
// api/src/Serializer/BookAttributeNormalizer.php
namespace App\Serializer;

use Symfony\Component\Security\Core\Authentication\Token\Storage\TokenStorageInterface;
use Symfony\Component\Serializer\Normalizer\ContextAwareNormalizerInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerAwareInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerAwareTrait;

class BookAttributeNormalizer implements ContextAwareNormalizerInterface, NormalizerAwareInterface
{
    use NormalizerAwareTrait;

    private const ALREADY_CALLED = 'BOOK_ATTRIBUTE_NORMALIZER_ALREADY_CALLED';

    private $tokenStorage;

    public function __construct(TokenStorageInterface $tokenStorage)
    {
        $this->tokenStorage = $tokenStorage;
    }

    public function normalize($object, $format = null, array $context = [])
    {
        if ($this->userHasPermissionsForBook($object)) {
            $context['groups'][] = 'can_retrieve_book';
        }

        $context[self::ALREADY_CALLED] = true;

        return $this->normalizer->normalize($object, $format, $context);
    }

    public function supportsNormalization($data, $format = null, array $context = [])
    {
        // Убедитесь, что нас не вызывают дважды
        if (isset($context[self::ALREADY_CALLED])) {
            return false;
        }

        return $data instanceof Book;
    }

    private function userHasPermissionsForBook($object): bool
    {
        // Получаем разрешения от пользователя в $this->tokenStorage
        // для текущего объекта $object (book) и
        // вернем true or false
    }
}
```

Это добавит группу сериализации `can_retrieve_book` только в том случае, если текущий вошедший в систему пользователь имеет доступ к данному экземпляру книги.

Примечание: В этом примере мы используем `TokenStorageInterface` для проверки доступа к экземпляру книги. Однако Symfony
предоставляет множество других полезных сервисов, которые могут лучше подойти для вашего варианта использования. Например, в [`AuthorizationChecker`](https://symfony.com/doc/current/components/security/authorization.html#authorization-checker).

## Преобразование имени

Компонент Serializer предоставляет удобный способ сопоставления имен полей PHP с сериализованными именами. Смотрите соответствующую [документацию Symfony](http://symfony.com/doc/master/components/serializer.html#converting-property-names-when-serializing-and-deserializing).


Чтобы использовать эту функцию, объявите новую службу преобразования имен. Например, вы можете преобразовать `camelCase` в
`snake_case` со следующей конфигурацией:

```yaml
# api/config/services.yaml
services:
    'Symfony\Component\Serializer\NameConverter\CamelCaseToSnakeCaseNameConverter': ~
```

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    name_converter: 'Symfony\Component\Serializer\NameConverter\CamelCaseToSnakeCaseNameConverter'
```

If symfony's `MetadataAwareNameConverter` is available it'll be used by default. If you specify one in ApiPlatform configuration, it'll be used. Note that you can use decoration to benefit from this name converter in your own implementation.

Если доступен `MetadataAwareNameConverter`, он будет использоваться по умолчанию. Так же будет использоваться тот который вы укажите в кофигурации Api Platform. Обратите внимание, что вы можете реализовать собственную реализацию преоброзователя имен обернув логику своим декоратором.

## Декорирование сериализатора и добавление дополнительных данных


В следующем примере мы увидим, как мы добавляем дополнительную информацию в сериализованный вывод. Вот как мы добавляем дату каждого запроса в `GET`:

```yaml
# api/config/services.yaml
services:
    'App\Serializer\ApiNormalizer':
        # По умолчанию .inner передается как аргумент
        decorates: 'api_platform.jsonld.normalizer.item'
```

Примечание: этот нормализатор будет работать только для формата JSON-LD, если вы хотите обрабатывать и данные JSON, вам нужно декорировать другой сервис:

```yaml
    # Нужно другое имя, чтобы избежать дублирования ключа YAML
    'app.serializer.normalizer.item.json':
        class: 'App\Serializer\ApiNormalizer'
        decorates: 'api_platform.serializer.normalizer.item'
```

```php
<?php
// api/src/Serializer/ApiNormalizer
namespace App\Serializer;

use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;
use Symfony\Component\Serializer\SerializerAwareInterface;
use Symfony\Component\Serializer\SerializerInterface;

final class ApiNormalizer implements NormalizerInterface, DenormalizerInterface, SerializerAwareInterface
{
    private $decorated;

    public function __construct(NormalizerInterface $decorated)
    {
        if (!$decorated instanceof DenormalizerInterface) {
            throw new \InvalidArgumentException(sprintf('Декорированный нормализатор должен реализовывать %s.', DenormalizerInterface::class));
        }

        $this->decorated = $decorated;
    }

    public function supportsNormalization($data, $format = null)
    {
        return $this->decorated->supportsNormalization($data, $format);
    }

    public function normalize($object, $format = null, array $context = [])
    {
        $data = $this->decorated->normalize($object, $format, $context);
        if (is_array($data)) {
            $data['date'] = date(\DateTime::RFC3339);
        }

        return $data;
    }

    public function supportsDenormalization($data, $type, $format = null)
    {
        return $this->decorated->supportsDenormalization($data, $type, $format);
    }

    public function denormalize($data, string $type, string $format = null, array $context = [])
    {
        return $this->decorated->denormalize($data, $type, $format, $context);
    }

    public function setSerializer(SerializerInterface $serializer)
    {
        if($this->decorated instanceof SerializerAwareInterface) {
            $this->decorated->setSerializer($serializer);
        }
    }
}
```

## Случай идентификатора сущности (Entity Identifier Case)

API Platform способна угадать идентификатор объекта, используя метаданные Doctrine ([ORM](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/basic-mapping.html#identifiers-primary-keys) , [MongoDB ODM](https://www.doctrine-project.org/projects/doctrine-mongodb-odm/en/latest/reference/basic-mapping.html#identifiers)).
Для ORM также поддерживается [составной identifiers](https://www.doctrine-project.org/projects/doctrine-orm/en/current/tutorials/composite-primary-keys.html).

Если вы не используете Doctrine ORM или MongoDB ODM провайлеры, вы должны явно пометить идентификатор, используя атрибут `identifier`
аннотации `ApiPlatform\Metadata\ApiProperty`.
Например:
```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\ApiProperty;

#[ApiResource]
class Book
{
    // ...

    #[ApiProperty(identifier: true)]
    private $id;

    /**
     * Этим полем может управлять только администратор
     */
    public bool $active = false;

    /**
     * Этим полем может управлять любой пользователь
     */
    public string $name;

    // ...
}
```

Вы также можете использовать формат конфигурации YAML:

```yaml
# api/config/api_platform/resources.yaml
properties:
    App\Entity\Book:
        id:
            identifier: true
```

In some cases, you will want to set the identifier of a resource from the client (e.g. a client-side generated UUID, or a slug).
In such cases, you must make the identifier property a writable class property. Specifically, to use client-generated IDs, you
must do the following:

1. create a setter for the identifier of the entity (e.g. `public function setId(string $id)`) or make it a `public` property ,
2. add the denormalization group to the property (only if you use a specific denormalization group), and,
3. if you use Doctrine ORM, be sure to **not** mark this property with [the `@GeneratedValue` annotation](http://docs.doctrine-project.org/projects/doctrine-orm/en/latest/reference/basic-mapping.html#identifier-generation-strategies)
  or use the `NONE` value

В некоторых случаях вам захочется задать идентификатор ресурса от клиента (например, сгенерированный на стороне клиента UUID или slug).
В таких случаях вы должны сделать свойство класса identifier доступным для записи.
В частности, чтобы использовать идентификаторы, сгенерированные клиентом, вы
должны выполнить следующее:

1. создайте сеттер для идентификатора объекта (например, `public function setId(string $id)`) или сделайте его свойством `public`
2. добавьте группу денормализации в свойство (только если вы используете определенную группу денормализации) и
3. если вы используете Doctrine ORM, убедитесь что **не** пометили это свойство с помощью [`@GeneratedValue` annotation](http://docs.doctrine-project.org/projects/doctrine-orm/en/latest/reference/basic-mapping.html#identifier-generation-strategies)
  или используйте значение `NONE`

## Встраивание контекста JSON-LD

По умолчанию сгенерированный [контекст JSON-LD](https://www.w3.org/TR/json-ld/#the-context ) (`@context`) ссылается только
IRI. Клиент, использующий JSON-LD, для его получения, должен отправить второй HTTP-запрос:

```json
{
  "@context": "/contexts/Book",
  "@id": "/books/62",
  "@type": "Book",
  "name": "My awesome book",
  "author": "/people/59"
}
```

Вы можете настроить API Platform для встраивания контекста JSON-LD в корневой документ, добавив атрибут `jsonld_embed_context` в аннотацию `#[ApiResource]`:

[codeSelector]

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;

#[ApiResource(normalizationContext: ['jsonld_embed_context' => true])]
class Book
{
    // ...
}
```

```yaml
# api/config/api_platform/resources/Book.yaml
App\Entity\Book:
    normalizationContext:
        jsonldEmbedContext: true
```

[/codeSelector]


Вывод JSON теперь будет включать встроенный контекст:

```json
{
  "@context": {
    "@vocab": "http://localhost:8000/apidoc#",
    "hydra": "http://www.w3.org/ns/hydra/core#",
    "name": "https://schema.org/name",
    "author": "https://schema.org/author"
  },
  "@id": "/books/62",
  "@type": "Book",
  "name": "My awesome book",
  "author": "/people/59"
}
```

## Отношение коллекций (Collection Relation)

Это особый случай, когда в сущности у вас есть отношение toMany. По умолчанию Doctrine будет использовать `ArrayCollection` для хранения ваших значений. Это нормально, когда у вас есть операция *чтения*, но когда вы пытаетесь *записать*, вы можете наблюдать проблему, когда ответ неправильно отражает изменения. Это может привести к ошибкам клиента, даже если обновление было правильным.
Действительно, после обновления этого отношения коллекция выглядит неправильно, потому что индексы `ArrayCollection` не являются последовательными. Чтобы изменить это, мы рекомендуем использовать геттер, который возвращает $collectionRelation->getValues(). Благодаря этому отношение теперь представляет собой настоящий массив, который последовательно индексируется.

```php
<?php
// api/src/Entity/Brand.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource]
final class Brand
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    #[ORM\ManyToMany(targetEntity: Car::class, inversedBy: 'brands')]
    #[ORM\JoinTable(name: 'CarToBrand')]
    #[ORM\JoinColumn(name: 'brand_id', referencedColumnName: 'id', nullable: false)]
    #[ORM\InverseJoinColumn(name: 'car_id', referencedColumnName: 'id', nullable: false)]
    private $cars;

    public function __construct()
    {
        $this->cars = new ArrayCollection();
    }

    public function addCar(DummyCar $car)
    {
        $this->cars[] = $car;
    }

    public function removeCar(DummyCar $car)
    {
        $this->cars->removeElement($car);
    }

    public function getCars()
    {
        return $this->cars->getValues();
    }

    public function getId()
    {
        return $this->id;
    }
}
```

Для справки, пожалуйста, просмотрите [#1534](https://github.com/api-platform/core/pull/1534).
