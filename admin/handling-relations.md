# Handling Relations

API Platform Admin handles `to-one` and `to-many` relations automatically.

If the relation is [embedded](serialization.md#embedding-relations), the admin automatically replaces the embedded resource data by their(s) IRI.
However, the embedded data is inserted to a local cache: it will not be necessary to make more requests if you reference some fields of the embedded resource later on.

Thanks to [the Schema.org support](schema.org.md), you can easily display the name of a related resource instead of its IRI.

## Display a Field of an Embedded Relation

If you use [embedded relations](serialization.md#embedding-relations) and need to display a nested field, make sure you write the code as if the relation needs to be fetched as a reference.

You *cannot* use the dot separator to do so.

For instance, if your API returns:

```json
{
  "@context": "/contexts/Book",
  "@id": "/books",
  "@type": "hydra:Collection",
  "hydra:member": [
    {
      "@id": "/books/07b90597-542e-480b-a6bf-5db223c761aa",
      "@type": "http://schema.org/Book",
      "title": "War and Peace",
      "author": {
        "@id": "/authors/d7a133c1-689f-4083-8cfc-afa6d867f37d",
        "@type": "http://schema.org/Author",
        "firstName": "Leo",
        "lastName": "Tolstoi"
      }
    }
  ],
  "hydra:totalItems": 1
}
```

If you want to display the author first name in the list, you need to write the following code:

```javascript
import React from "react";
import {
  HydraAdmin,
  FieldGuesser,
  ListGuesser,
  ResourceGuesser
} from "@api-platform/admin";
import { ReferenceField, TextField } from "react-admin";

const BooksList = (props) => (
  <ListGuesser {...props}>
    <FieldGuesser source="title" />
    {/* Use react-admin components directly when you want complex fields. */}
    <ReferenceField label="Author first name" source="author" reference="authors">
      <TextField source="firstName" />
    </ReferenceField>
  </ListGuesser>
);

export default () => (
  <HydraAdmin entrypoint={process.env.REACT_APP_API_ENTRYPOINT}>
    <ResourceGuesser
      name="books"
      list={BooksList}
    />
  </HydraAdmin>
);
```

## Using an Autocomplete Input for Relations

Let's go one step further thanks to the [customization capabilities](customizing.md) of API Platform Admin by adding autocompletion support to form inputs for relations.

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
