# Handling Relations to Collections

Currently, API Platform Admin doesn't handle `to-many` relations. The core library [is being patched](https://github.com/api-platform/core/pull/1189)
to document relations to collections through OWL.

Meanwhile, it is possible to manually configure API Platform to handle relations to collections.

We will create the admin for an API exposing `Person` and `Book` resources linked with a `many-to-many`
relation between them (through the `authors` property).

This API can be created using the following PHP code:

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

Let's customize the components used for the `authors` property:

```javascript
import React, { Component } from 'react';
import { ReferenceArrayField, SingleFieldList, ChipField, ReferenceArrayInput, SelectArrayInput } from 'react-admin';
import { AdminBuilder, hydraClient } from '@api-platform/admin';
import parseHydraDocumentation from 'api-doc-parser/lib/hydra/parseHydraDocumentation';

const entrypoint = 'https://demo.api-platform.com';

export default class extends Component {
  state = {api: null, resources: null};

  componentDidMount() {
    parseHydraDocumentation(entrypoint).then({api, resources} => {
        const books = r.find(r => 'books' === r.name);

        // Set the field in the list and the show views
        books.readableFields.find(f => 'authors' === f.name).fieldComponent =
          <ReferenceArrayField label="Authors" reference="people" source="authors" key="authors">
            <SingleFieldList>
              <ChipField source="name" key="name"/>
            </SingleFieldList>
          </ReferenceArrayField>
        ;

        // Set the input in the edit and create views
        books.writableFields.find(f => 'authors' === f.name).inputComponent =
          <ReferenceArrayInput label="Authors" reference="people" source="authors" key="authors">
            <SelectArrayInput optionText="name"/>
          </ReferenceArrayInput>
        ;

        this.setState({api, resources});
      }
    )
  }

  render() {
    if (null === this.state.api) return <div>Loading...</div>;

    return <AdminBuilder api={this.state.api} dataProvider={hydraClient({entrypoint: entrypoint, resources: this.state.resources})}/>
  }
}
```

The admin now properly handles this `to-many` relation!

## Using an Autocomplete Input for Relations

We'll make one last improvement to our admin: transforming the relation selector we just created to use autocompletion.

Start by adding a "partial search" filter on the `name` property of the `Book` resource class.

```yaml
# api/config/services.yaml
services:
    person.search_filter:
        parent: 'api_platform.doctrine.orm.search_filter'
        arguments: [ { name: 'partial' } ]
        # Uncomment only if you don't use autoconfiguration
        #tags: ['api_platform.filter']
```

```php
<?php
// api/src/Entity/Person.php
// ...

/**
 * @ApiResource(attributes={"filters"={"person.search_filter"}})
 * @ORM\Entity
 */
class Person
{
    // ...
}
```

Then edit the configuration of API Platform Admin to pass a `filterToQuery` property to the `ReferenceArrayInput` component.

```javascript
  componentDidMount() {

    // ...

    // Set the input in the edit and create views
    books.writableFields.find(f => 'authors' === f.name).inputComponent =
      <ReferenceArrayInput label="Authors" reference="people" source="authors" key="authors" filterToQuery={searchText => ({ name: searchText })}>
        <SelectArrayInput optionText="name"/>
      </ReferenceArrayInput>
    ;

    // ...
  }
```

The autocomplete field should now work properly!
