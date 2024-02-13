# Subresources

A Subresource is another way of declaring a resource that usually involves a more complex URI.
In API Platform you can declare as many `ApiResource` as you want on a PHP class
creating Subresources.

Subresources work very well by implementing your own state [providers](./state-providers.md)
or [processors](./state-processors.md). In API Platform we provide a working Doctrine layer for
subresources providing you add the correct configuration for URI Variables.

## URI Variables Configuration

URI Variables are configured via the `uriVariables` node on an `ApiResource`. It's an array indexed by the variables present in your URI, `/companies/{companyId}/employees/{id}` has two uri variables `companyId` and `id`. For each of these, we need to create a `Link` between the previous and the next node, in this example the link between a Company and an Employee.

If you're using the Doctrine implementation, queries are automatically built using the provided links.

### Answer to a Question

For this example we have two classes, a Question and an Answer. We want to find the Answer to
the Question about the Universe using the following URI: `/question/42/answer`.

Let's start by defining the resources:

<code-selector>

```php
<?php
// api/src/Entity/Answer.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
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

use ApiPlatform\Metadata\ApiResource;
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
    public Answer $answer;

    public function getId(): ?int
    {
        return $this->id;
    }

    // ...
}
```

```yaml
# api/config/api_platform/resources.yaml
resources:
    App\Entity\Answer: ~
    App\Entity\Question: ~
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
        https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Question"/>
    <resource class="App\Entity\Answer"/>

</resources>
```

</code-selector>

Now to create a new way of retrieving an Answer we will declare another resource on the `Answer` class.
To make things work, API Platform needs information about how to retrieve the `Answer` belonging to
the `Question`, this is done by configuring the `uriVariables`:

<code-selector>

```php
<?php
// api/src/Entity/Answer.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Link;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource]
#[ApiResource(
    uriTemplate: '/questions/{id}/answer', 
    uriVariables: [
        'id' => new Link(
            fromClass: Question::class,
            fromProperty: 'answer'
        )
    ], 
    operations: [new Get()]
)]
class Answer
{
    // ...
}
```

```yaml
# api/config/api_platform/resources.yaml
resources:
    App\Entity\Answer:
        uriTemplate: /questions/{id}/answer
        uriVariables:
            id:
                fromClass: App\Entity\Question
                fromProperty: answer
        operations:
            ApiPlatform\Metadata\Get: ~

    App\Entity\Question: ~
```

```xml
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
        https://api-platform.com/schema/metadata/resources-3.0.xsd">
    
    <resource class="App\Entity\Question"/>

    <resource class="App\Entity\Answer"/>

    <resource class="App\Entity\Answer" uriTemplate="/questions/{id}/answer">
        <uriVariables>
            <uriVariable parameterName="id" fromClass="App\Entity\Question" fromProperty="answer"/>
        </uriVariables>

        <operations>
            <operation class="ApiPlatform\Metadata\Get"/>
        </operations>
    </resource>
</resources>    

```

</code-selector>

In this example, we instructed API Platform that the `Answer` we retrieve comes **from** the **class** `Question`
**from** the **property** `answer` of that class.

URI Variables are defined using Links (`ApiPlatform\Metadata\Link`). A `Link` can be binded either from or to a class and a property.

If we had a `relatedQuestions` property on the `Answer` we could retrieve the collection of related questions via the following definition:

<code-selector>

```php
#[ApiResource(
    uriTemplate: '/answers/{id}/related_questions.{_format}',
    uriVariables: [
        'id' => new Link(fromClass: Answer::class, fromProperty: 'relatedQuestions')
    ], 
    operations: [new GetCollection()]
)]
```

```yaml
# api/config/api_platform/resources.yaml
resources:
    App\Entity\Question:
        uriTemplate: /answers/{id}/related_questions.{_format}
        uriVariables:
            id:
                fromClass: App\Entity\Answer
                fromProperty: relatedQuestions
        operations:
            ApiPlatform\Metadata\GetCollection: ~
```

```xml
<resource class="App\Entity\Question" uriTemplate="/answers/{id}/related_questions.{_format}">
    <uriVariables>
        <uriVariable parameterName="id" fromClass="App\Entity\Answer" fromProperty="relatedQuestions"/>
    </uriVariables>

    <operations>
        <operation class="ApiPlatform\Metadata\GetCollection"/>
    </operations>

</resource>
```

</code-selector>

### Company Employee's

Note that in this example, we declared an association using Doctrine only between Employee and Company using a ManyToOne. There is no inverse association hence the use of `toProperty` in the URI Variables definition.

The following declares a few subresources:
    - `/companies/{companyId}/employees/{id}` - get an employee belonging to a company
    - `/companies/{companyId}/employees` - get the company employee's

```php
<?php
// api/src/Entity/Employee.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Link;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\Post;
use ApiPlatform\Metadata\GetCollection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource(
    operations: [ new Post() ]
)]
#[ApiResource(
    uriTemplate: '/companies/{companyId}/employees/{id}',
    uriVariables: [
        'companyId' => new Link(fromClass: Company::class, toProperty: 'company'),
        'id' => new Link(fromClass: Employee::class),
    ],
    operations: [ new Get() ]
)]
#[ApiResource(
    uriTemplate: '/companies/{companyId}/employees',
    uriVariables: [
        'companyId' => new Link(fromClass: Company::class, toProperty: 'company'),
    ],
    operations: [ new GetCollection() ]
)]
class Employee
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    public ?int $id;

    #[ORM\Column]
    public string $name;

    #[ORM\ManyToOne(targetEntity: Company::class)]
    public ?Company $company;

    public function getId()
    {
        return $this->id;
    }
}
```

Now let's add the Company class:

```php
<?php
// api/src/Entity/Company.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\Link;
use ApiPlatform\Metadata\Post;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource]
class Company
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    public ?int $id;

    #[ORM\Column]
    public string $name;

    /** @var Employee[] */
    #[Link(toProperty: 'company')]
    public $employees = []; // only used to set metadata as GraphQl always needs to work from both sides of the association
}
```

We did not define any Doctrine annotation here and if we want things to work properly with GraphQL, we need to map the `employees` field as a Link to the class `Employee` using the property `company`.

As a general rule, if the property we want to create a link from is in the `fromClass`, use `fromProperty`, if not, use `toProperty`.

For example, we could add a subresource fetching an employee's company. The `company` property belongs to the `Employee` class we can use `fromProperty`:

```php
<?php 
#[ApiResource(
    uriTemplate: '/employees/{employeeId}/company',
    uriVariables: [
        'employeeId' => new Link(fromClass: Employee::class, fromProperty: 'company'),
    ],
    operations: [
        new Get()
    ]
)]

class Company {
    // ...
}
```
