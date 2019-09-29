# Handling Relations to Collections

API Platform Admin handles `to-one` and `to-many` relations automatically.
Thanks to [the Schema.org support](schema.org.md), it's also easy to display the name of a related objects instead of its IRI.

Let's go one step further thanks to the [customization capabilities](customizing.md) of API Platform Admin by adding autocompletion support to form inputs.

## Using an Autocomplete Input for Relations

Let's consider an API exposing `Person` and `Book` resources linked by a `many-to-many`
relation (through the `authors` property).

This API uses the following PHP code:

```php
<?php
// api/src/Entity/Person.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource
 * @ORM\Entity
 */
class Person
{
    /**
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue
     * @ORM\Id
     */
    public $id;

    /**
     * @ORM\Column
     * @ApiFilter(SearchFilter::class, strategy="ipartial")
     */
    public $name;
}
```

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource
 * @ORM\Entity
 */
class Book
{
    /**
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue
     * @ORM\Id
     */
    public $id;

    /**
     * @ORM\ManyToMany(targetEntity="Person")
     */
    public $authors;

    public function __construct()
    {
        $this->authors = new ArrayCollection();
    }
}
```

Notice the "partial search" [filter](../core/filters.md) on the `name` property of the `Book` resource class.

Now, let's configure API Platform Admin to enable autocompletion for the relation selector:

```javascript
import React from "react";
import {
  HydraAdmin,
  ResourceGuesser,
  CreateGuesser,
  EditGuesser,
  InputGuesser
} from "@api-platform/admin";
import { ReferenceInput, AutocompleteInput } from "react-admin";

const ReviewsCreate = props => (
  <CreateGuesser {...props}>
    <InputGuesser source="author" />
    <ReferenceInput
      source="book"
      reference="books"
      label="Books"
      filterToQuery={searchText => ({ title: searchText })}
    >
      <AutocompleteInput optionText="title" />
    </ReferenceInput>

    <InputGuesser source="rating" />
    <InputGuesser source="body" />
    <InputGuesser source="publicationDate" />
  </CreateGuesser>
);

const ReviewsEdit = props => (
  <EditGuesser {...props}>
    <InputGuesser source="author" />

    <ReferenceInput
      source="book"
      reference="books"
      label="Books"
      filterToQuery={searchText => ({ title: searchText })}
    >
      <AutocompleteInput optionText="title" />
    </ReferenceInput>

    <InputGuesser source="rating" />
    <InputGuesser source="body" />
    <InputGuesser source="publicationDate" />
  </EditGuesser>
);

export default () => (
  <HydraAdmin entrypoint={process.env.REACT_APP_API_ENTRYPOINT}>
    <ResourceGuesser
      name="reviews"
      create={ReviewsCreate}
      edit={ReviewsEdit}
    />
  </HydraAdmin>
);
```

The autocomplete field should now work properly!
