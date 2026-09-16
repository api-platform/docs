# Performance Tips

To make the admin faster and greener, you can make some changes to your API.

## Retrieve All Relations in One Request

Some of your relations are not embedded. You display some of their fields in your resource list. By
default, the admin fetches these relations one by one.

In this case, it can be improved by doing only one request for all the related resources instead.

To do so, you need to make sure the [`ExactFilter`](../core/doctrine-filters.md#exact-filter) is
enabled for the identifier of the related resource.

For instance, say you have a `book` resource with a relation to `author` resources, and you display
the author names on your book list. Retrieve the authors in one go by writing:

```php
<?php
// api/src/Entity/Author.php
namespace App\Entity;

use ApiPlatform\Doctrine\Orm\Filter\ExactFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\QueryParameter;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource(
    parameters: [
        'id' => new QueryParameter(filter: new ExactFilter()),
    ]
)]
class Author
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    public ?int $id = null;

    #[ORM\Column]
    public string $name;
}
```

The admin now fetches all the authors in one go, instead of issuing a separate request for each
author. The request looks similar to the following:

```txt
https://localhost/authors?
  page=1
  &itemsPerPage=5
  &id[0]=/authors/7
  &id[1]=/authors/8
  &id[2]=/authors/9
  &id[3]=/authors/10
  &id[4]=/authors/11
```
