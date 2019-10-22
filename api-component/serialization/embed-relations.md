# Embedding Relations

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/relations?cid=apip"><img src="../../distribution/images/symfonycasts-player.png" alt="Relations screencast"><br>Watch the Relations screencast</a></p>

By default, the serializer provided with API Platform represents relations between objects using [dereferenceable IRIs](https://en.wikipedia.org/wiki/Internationalized_Resource_Identifier).
They allow you to retrieve details for related objects by issuing extra HTTP requests.

In the following JSON document, the relation from a book to an author is represented by an URI:

```json
{
  "@context": "/contexts/Book",
  "@id": "/books/62",
  "@type": "Book",
  "name": "My awesome book",
  "author": "/people/59"
}
```

However, for performance reasons, it is sometimes preferable to avoid forcing the client to issue extra HTTP requests.
It is possible to embed related objects (in their entirety, or only some of their properties) directly in the parent
response through the use of serialization groups. By using the following serialization groups annotations (`@Groups`),
a JSON representation of the author is embedded in the book response:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource(normalizationContext={"groups"={"book"}})
 */
class Book
{
    /**
     * @Groups({"book"})
     */
    public $name;

    /**
     * @Groups({"book"})
     */
    public $author;

    // ...
}
```

```php
<?php
// api/src/Entity/Person.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource
 */
class Person
{
    /**
     * ...
     * @Groups("book")
     */
    public $name;

    // ...
}
```

The generated JSON using previous settings is below:

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

In order to optimize such embedded relations, the default Doctrine data provider will automatically join entities on relations
marked as [`EAGER`](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/annotations-reference.html#manytoone).
This avoids the need for extra queries to be executed when serializing the related objects.

Instead of embedding relations in the main HTTP response, you may want [to "push" them to the client using HTTP/2 server push](../usage-and-configuration/push-relations.md).

## Denormalizing relations

It is also possible to embed a relation in `PUT` and `POST` requests. To enable that feature, set the serialization groups
the same way as normalization. For example:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(denormalizationContext={"groups"={"book"}})
 */
class Book
{
    // ...
}
```

The following rules apply when denormalizing embedded relations:

* If an `@id` key is present in the embedded resource, then the object corresponding to the given URI will be retrieved through
the data provider. Any changes in the embedded relation will also be applied to that object.
* If no `@id` key exists, a new object will be created containing data provided in the embedded JSON document.

You can specify as many embedded relation levels as you want.



## Collection Relations

This is a special case where, in an entity, you have a `toMany` relation. By default, Doctrine will use an `ArrayCollection` to store your values. 

This is fine when you have a *read* operation, but when you try to *write* you can observe an issue where the response is not reflecting the changes correctly. 
It can lead to client errors even though the update was correct.

Indeed, after an update on this relation, the collection looks wrong because `ArrayCollection`'s indexes are not sequential. 
To change this, we recommend to use a getter that returns `$collectionRelation->getValues()`. Thanks to this, the relation is now a real array which is sequentially indexed.

```php
<?php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource
 * @ORM\Entity
 */
final class Brand
{
    /**
     * @ORM\Id
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    private $id;

    /**
     * @ORM\ManyToMany(targetEntity="App\Entity\Car", inversedBy="brands")
     * @ORM\JoinTable(
     *     name="CarToBrand",
     *     joinColumns={@ORM\JoinColumn(name="brand_id", referencedColumnName="id", nullable=false)},
     *     inverseJoinColumns={@ORM\JoinColumn(name="car_id", referencedColumnName="id", nullable=false)}
     * )
     */
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

For reference please check [#1534](https://github.com/api-platform/core/pull/1534).
