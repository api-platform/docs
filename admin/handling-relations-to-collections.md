# Handling Relations to Collections

Considering an API exposing `Person` and `Book` resources linked with a `many-to-many`
relation between them (through the `authors` property).

This API using the following PHP code:

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

The admin handles this `to-many` relation automatically!

## Customizing a Property

Let's customize the components used for the `authors` property, to display them by their 'name' instead 'id' (the default behavior).

```javascript
import React, { Component } from 'react';
import { ReferenceArrayField, SingleFieldList, ChipField, ReferenceArrayInput, SelectArrayInput } from 'react-admin';
import { AdminBuilder, hydraClient } from '@api-platform/admin';
import parseHydraDocumentation from '@api-platform/api-doc-parser/lib/hydra/parseHydraDocumentation';

const entrypoint = 'https://demo.api-platform.com';

export default class extends Component {
  state = { api: null }

  componentDidMount() {
    parseHydraDocumentation(entrypoint).then(({api}) => {
        const books = api.resources.find(({ name }) => 'books' === name)
        const authors = books.fields.find(({ name }) => 'authors' === name)

        // Set the field in the list and the show views
        authors.field = props => (
          <ReferenceArrayField source={authors.name} reference={authors.reference.name} key={authors.name} {...props}>
            <SingleFieldList>
              <ChipField source="name" key="name"/>
            </SingleFieldList>
          </ReferenceArrayField>
        );

        // Set the input in the edit and create views
        authors.input = props => (
          <ReferenceArrayInput source={authors.name} reference={authors.reference.name} label="Authors" key={authors.name} {...props} allowEmpty>
            <SelectArrayInput optionText="name"/>
          </ReferenceArrayInput>
        );

        this.setState({ api });
      }
    )
  }

  render() {
    if (null === this.state.api) return <div>Loading...</div>;

    return <AdminBuilder api={ this.state.api } dataProvider={ hydraClient(this.state.api) }/>
  }
}
```


## Customizing an Icon

Now that our `authors` property is displaying the name instead of an 'id', let's change the icon shown in the list menu.

Just add an import statement from `@material-ui` for adding the icon, in this case, a user icon:

`import UserIcon from '@material-ui/icons/People';`

and add it to the `authors.icon` property

The code for just customizing the icon will be:

```javascript
import React, { Component } from 'react';
import { AdminBuilder, hydraClient } from '@api-platform/admin';
import parseHydraDocumentation from '@api-platform/api-doc-parser/lib/hydra/parseHydraDocumentation';
import UserIcon from '@material-ui/icons/People';

const entrypoint = 'https://demo.api-platform.com';

export default class extends Component {
  state = { api: null }

  componentDidMount() {
    parseHydraDocumentation(entrypoint).then(({api}) => {
        const authors = books.fields.find(({ name }) => 'authors' === name)

        // Set the icon
        authors.icon = UserIcon
       
        this.setState({ api });
      }
    )
  }

  render() {
    if (null === this.state.api) return <div>Loading...</div>;

    return <AdminBuilder api={ this.state.api } dataProvider={ hydraClient(this.state.api) }/>
  }
}
```

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
      authors.input = props => (
        <ReferenceArrayInput source={authors.name} reference={authors.reference.name} label="Authors" key={authors.name} filterToQuery={searchText => ({ name: searchText })} {...props} allowEmpty>
          <SelectArrayInput optionText="name"/>
        </ReferenceArrayInput>
      );

    // ...
  }
```

The autocomplete field should now work properly!
