# Performance

To make the admin faster and greener, you can make some changes to your API.

## Retrieve All Relations in One Request

By default, if your relations are not embedded and if you decide to display some fields belonging to relations in your resource list,
the admin will fetch the relations one by one.

In this case, it can be improved by doing only one request for all the related resources instead.

To do so, you need to make sure the [search filter](../core/filters.md#search-filter) is enabled for the identifier of the related resource.

For instance, if you have a book resource having a relation to author resources and you display the author names on your book list,
you can make sure the authors are retrieved in one go by writing:

```php
<?php
// api/src/Entity/Author.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource
 * @ORM\Entity
 */
class Author
{
    /**
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue
     * @ORM\Id
     * @ApiFilter(SearchFilter::class, strategy="exact")
     */
    public $id;

    /**
     * @ORM\Column
     */
    public $name;
}
```
