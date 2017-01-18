# Validation

API Platform Core uses the [Symfony Validator component](http://symfony.com/doc/current/book/validation.html) to validate
entities.

Without specific configuration, it uses the default validation group, but this behavior is customizable.

## Using Validation Groups

Built-in actions are able to leverage Symfony's [validation groups](http://symfony.com/doc/current/book/validation.html#validation-groups).

You can customize them by editing the resource configuration and add the groups you want to use when the validation occurs:

```php
<?php

// src/AppBundle/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * @ApiResource(attributes={"validation_groups"={"a", "b"}})
 */
class Book
{
    /**
     * @Assert\NotBlank(groups={"a"})
     */
    private $name;

    /**
     * @Assert\NotNull(groups={"b"})
     */
    private $author;

    // ...
}
```

With the previous configuration, the validations groups `a` and `b` will be used when validation is performed.

Like for [serialization groups](serialization-groups-and-relations.md#using-different-serialization-groups-per-operation),
you can specify validation groups globally or on a per operation basis.

Of course, you can use XML or YAML configuration format instead of annotations if you prefer.

You may also pass in a [group sequence](http://symfony.com/doc/current/validation/sequence_provider.html) in place of
the array of group names.

## Dynamic Validation Groups

If you need to dynamically determine which validation groups to use for an entity in different scenarios, just pass in a
[callable](http://php.net/manual/en/language.types.callable.php). The callback will receive the entity object as its first
argument, and should return an array of group names or a [group sequence](http://symfony.com/doc/current/validation/sequence_provider.html).

In the following example, we use a static method to return the validation groups:

```php
<?php

// src/AppBundle/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * @ApiResource(attributes={"validation_groups"={Book::class, "validationGroups"}})
 */
class Book
{
    /**
     * Return dynamic validation groups.
     *
     * @param self $book Contains the instance of Book to validate.
     *
     * @return string[]
     */
    public static function validationGroups(self $book)
    {
        return ['a'];
    }

    /**
     * @Assert\NotBlank(groups={"a"})
     */
    private $name;

    /**
     * @Assert\NotNull(groups={"b"})
     */
    private $author;

    // ...
}
```

Previous chapter: [Serialization Groups and Relations](serialization-groups-and-relations.md)

Next chapter: [Pagination](pagination.md)
