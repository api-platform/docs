# Handling Relations

API Platform Admin handles all types of relations (including `to-many` ones automatically!

Let's consider an API exposing `Person` and `Book` resources linked by a `many-to-many`
relation (through the `authors` property).

This API uses the following PHP code:

```php
<?php
// api/src/Entity/Person.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
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
     */
    public $name;
}
```

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
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

You have nothing to do! API Platform Admin is smart enough to deal with relations.

## Show the Names of your Entities Instead of their IRIs

See [how to use the Schema.org vocabulary](schema.org.md).

## Using an Autocomplete Input

A nice improvement to our admin could be to use an autocomplete input to select a pick a related object.

Server-side, start by adding a "partial search" filter on the `name` property of the `Book` resource class.

```php
<?php
// api/src/Entity/Person.php
// ...

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter

class Person
{
    /**
     * @ApiFilter(SearchFilter::class, strategy="ipartial")
     * ...
     */
    public $title;

    // ...
}
```

Then [customize API Platform Admin](customize.md) by using [the `<AutocompleteInput>` component provided by React Admin](https://marmelab.com/react-admin/Inputs.html#autocompleteinput), and configure it to leverage this new filter:

```javascript
import React from "react";
import {
  HydraAdmin,
  ResourceGuesser,
  CreateGuesser,
  InputGuesser
} from "@api-platform/admin";
import { ReferenceInput, AutocompleteInput } from "react-admin";

const ReviewsCreate = props => (
  <CreateGuesser {...props}>
    <ReferenceInput
      source="book"
      reference="books"
      label="Books"
      filterToQuery={searchText => ({ title: searchText })}
    >
      <AutocompleteInput optionText="title" />
    </ReferenceInput>
    {/* ... */}
  </CreateGuesser>
);

export default () => (
  <HydraAdmin entrypoint="https://demo.api-platform.com">
    <ResourceGuesser name="reviews" create={ReviewsCreate} />
    {/* ... */}
  </HydraAdmin>
);
```

And it works!
