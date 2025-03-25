# Performance Tips

To make the admin faster and greener, you can make some changes to your API.

## Retrieve All Relations in One Request

By default, if your relations are not embedded and if you decide to display some fields belonging to relations in your resource list,
the admin will fetch the relations one by one.

In this case, it can be improved by doing only one request for all the related resources instead.

To do so, you need to make sure the [search filter](../core/doctrine-filters.md#search-filter) is enabled for the identifier of the related resource.

For instance, if you have a `book` resource having a relation to `author` resources and you display the author names on your book list,
you can make sure the authors are retrieved in one go by writing:

```php
<?php
// api/src/Entity/Author.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;
use ApiPlatform\Metadata\ApiResource;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource]
class Author
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    #[ApiFilter(SearchFilter::class, strategy: "exact")]
    public ?int $id = null;

    #[ORM\Column]
    public string $name;
}
```

Instead of issuing a separate request for each author, the admin will now fetch all the authors in one go, with a request similar to the following:

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
