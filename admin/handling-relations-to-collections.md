# Handling Relations to Collections

Currently, API Platform Admin doesn't handle `to-many` relations. The core library [is being patched](https://github.com/api-platform/core/pull/1189)
to document relations to collections through OWL.

During the meantime, it is possible to configure manually API Platform to handle relations to collections.

We will create the admin for an API exposing `Person` and `Book` resources linked with a `many-to-many`
relation between them (trough the `authors` property).

This API can be created using the following PHP code:

```php
<?php

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
import { ReferenceArrayField, SingleFieldList, ChipField, ReferenceArrayInput, SelectArrayInput } from 'admin-on-rest';
import { AdminBuilder, hydraClient } from 'api-platform-admin';
import parseHydraDocumentation from 'api-doc-parser/lib/hydra/parseHydraDocumentation';

const entrypoint = 'https://demo.api-platform.com';

class Admin extends Component {
  state = {api: null};

  componentDidMount() {
    parseHydraDocumentation(entrypoint).then(api => {
        const r = api.resources;

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

        this.setState({api: api});
      }
    )
  }

  render() {
    if (null === this.state.api) return <div>Loading...</div>;

    return <AdminBuilder api={this.state.api} restClient={hydraClient(entrypoint)}/>
  }
}

export default Admin;
```

The admin now properly handle this `to-many` relation!

## Using an Autocomplete Input for Relations

We'll do a last improvement to our admin: transform the relation selector we just created to use autocompletion.

Start by adding a "partial search" filter on the `name` property of the `Book` resource class.

```yaml
# etc/packages/api_platform.yaml

services:
    person.search_filter:
        parent: 'api_platform.doctrine.orm.search_filter'
        arguments: [ { name: 'partial' } ]
        tags: ['api_platform.filter']
```

```php

// ...

/**
 * @ApiResource(attributes={"filters"={"person.search_filter"}})
 * @ORM\Entity
 */
class Person
{
// ...
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

Previous chapter: [Authentication Support](authentication-support.md)

Next chapter: [The Client Generator Component: Introduction](../client-generator/index.md)
