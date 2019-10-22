# Specify Entity Identifier

API Platform is able to guess the entity identifier using Doctrine metadata ([ORM](http://doctrine-orm.readthedocs.org/en/latest/reference/basic-mapping.html#identifiers-primary-keys), [MongoDB ODM](https://www.doctrine-project.org/projects/doctrine-mongodb-odm/en/latest/reference/basic-mapping.html#identifiers)).
For ORM, it also supports composite identifiers.

If you are not using the Doctrine ORM or MongoDB ODM Provider, you must explicitly mark the identifier using the `identifier` attribute of
the `ApiPlatform\Core\Annotation\ApiProperty` annotation. For example:


```php
/**
 * @ApiResource()
 */
class Book
{
    // ...

    /**
     * @ApiProperty(identifier=true)
     */
    private $id;

    /**
     * This field can be managed only by an admin
     *
     * @var bool
     */
    public $active = false;

    /**
     * This field can be managed by any user
     *
     * @var string
     */
    public $name;

    // ...
}
```

You can also use the YAML configuration format:

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Book:
    properties:
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
