# Subresources

A subresource is a collection or an item that belongs to another resource.
API Platform makes it easy to create such operations.

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/subresources?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Subresources screencast"><br>Watch the Subresources screencast</a></p>

The starting point of a subresource must be a relation on an existing resource.
For example, let's create two entities (Question, Answer) and set up a subresource so that `/question/42/answer` gives us
the answer to the question 42:

<code-selector>

```php
<?php
// api/src/Entity/Answer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource]
class Answer
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    #[ORM\Column(type: 'text')]
    public string $content;

    #[ORM\OneToOne]
    public Question $question;

    public function getId(): ?int
    {
        return $this->id;
    }

    // ...
}

// api/src/Entity/Question.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Annotation\ApiSubresource;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource]
class Question
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    #[ORM\Column(type: 'text')]
    public string $content;

    #[ORM\OneToOne]
    #[ORM\JoinColumn(referencedColumnName: 'id', unique: true)]
    #[ApiSubresource]
    public Answer $answer;

    public function getId(): ?int
    {
        return $this->id;
    }

    // ...
}
```

```yaml
# config/api_platform/resources.yaml
App\Entity\Answer: ~
App\Entity\Question:
    properties:
        answer:
            subresource:
                resourceClass: 'App\Entity\Answer'
                collection: false
```

</code-selector>

Note that all we had to do is to set up `#[ApiSubresource]` on the `Question::answer` relation. Because the `answer` is a to-one relation, we know that this subresource is an item. Therefore the response will look like this:

```json
{
  "@context": "/contexts/Answer",
  "@id": "/answers/42",
  "@type": "Answer",
  "id": 42,
  "content": "Life, the Universe, and Everything",
  "question": "/questions/42"
}
```

If you put the subresource on a relation that is to-many, you will retrieve a collection.

Last but not least, subresources can be nested, such that `/questions/42/answer/comments` will get the collection of comments for the answer to question 42.

Note: only for `GET` operations are supported at the moment

## Using Serialization Groups

You may want custom groups on subresources, you can set `normalization_context` or `denormalization_context` on that operation. To do so, add a `subresourceOperations` node. For example:

<code-selector>

```php
<?php
// api/src/Entity/Answer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

 #[ApiResource(
    subresourceOperations: [
        'api_questions_answer_get_subresource' => [
            'method' => 'GET',
            'normalization_context' => [
                'groups' => ['foobar'],
            ],
        ],
    ],
)]
class Answer
{
    // ...
}
```

```yaml
# config/api_platform/resources.yaml
App\Entity\Answer:
    subresourceOperations:
        api_questions_answer_get_subresource:
            method: 'GET'
            normalization_context: {groups: ['foobar']}
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata
           https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="App\Entity\Answer">
        <subresourceOperations>
            <subresourceOperation name="api_questions_answer_get_subresource">
                <attribute name="method">GET</attribute>
                <attribute name="normalization_context">
                  <attribute name="groups">
                    <attribute>foobar</attribute>
                  </attribute>
                </attribute>
            </subresourceOperation>
        </subresourceOperations>
    </resource>
</resources>
```

</code-selector>

In the previous examples, the `method` attribute is mandatory, because the operation name doesn't match a supported HTTP
method.

Note that the operation name, here `api_questions_answer_get_subresource`, is the important keyword.
It'll be automatically set to `$resources_$subresource(s)_get_subresource`. To find the correct operation name you
may use `bin/console debug:router`.

## Using Custom Paths

You can control the path of subresources with the `path` option of the `subresourceOperations` parameter.

```php
<?php
// api/src/Entity/Question.php

#[ApiResource(
    subresourceOperations: [
        'answer_get_subresource' => [
            'method' => 'GET',
            'path' => '/questions/{id}/all-answers',
        ],
    ],
)]
class Question
{
}
```

### Access Control of Subresources

The `subresourceOperations` attribute also allows you to add an access control on each path with the attribute `security`.

```php
<?php
// api/src/Entity/Answer.php

 #[ApiResource(
    subresourceOperations: [
        'api_questions_answer_get_subresource' => [
            'security' => "is_granted('ROLE_AUTHENTICATED')",
        ],
    ],
 )]
 class Answer
 {
 }
```

### Limiting Depth

You can control depth of subresources with the parameter `maxDepth`. For example, if the `Answer` entity also has a subresource
such as `comments` and you don't want the route `api/questions/{id}/answers/{id}/comments` to be generated. You can do this by adding the parameter maxDepth in the ApiSubresource annotation or YAML/XML file configuration.

```php
<?php
// api/src/Entity/Question.php

use ApiPlatform\Core\Annotation\ApiProperty;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Annotation\ApiSubresource;

#[ApiResource]
class Question
{
    #[ApiSubresource(
        maxDepth: 1,
    )]
    public $answer;

    // ...
}
```
