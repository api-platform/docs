# Security

To add a security layer to your queries and mutations, follow the [security](../security/security.md) documentation.

If your security needs differ between REST and GraphQL, add the particular parts in the `graphql` key.

In the example below, we want the same security rules as we have in REST, but we also want to allow an admin to delete a book only in GraphQL.
Please note that, it's not possible to update a book in GraphQL because the `update` operation is not defined.

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(
 *     attributes={"security"="is_granted('ROLE_USER')"},
 *     collectionOperations={
 *         "post"={"security"="is_granted('ROLE_ADMIN')", "security_message"="Only admins can add books."}
 *     },
 *     itemOperations={
 *         "get"={"security"="is_granted('ROLE_USER') and object.owner == user", "security_message"="Sorry, but you are not the book owner."}
 *     },
 *     graphql={
 *         "item_query"={"security"="is_granted('ROLE_USER') and object.owner == user"},
 *         "collection_query"={"security"="is_granted('ROLE_ADMIN')"},
 *         "delete"={"security"="is_granted('ROLE_ADMIN')"},
 *         "create"={"security"="is_granted('ROLE_ADMIN')"}
 *     }
 * )
 */
class Book
{
    // ...
}
```
