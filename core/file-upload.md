# Handling File Upload

As common a problem as it may seem, handling file upload requires a custom
implementation in your app. This page will guide you in handling file upload in
your API, with the help of
[VichUploaderBundle](https://github.com/dustin10/VichUploaderBundle). It is
recommended you [read the documentation of
VichUploaderBundle](https://github.com/dustin10/VichUploaderBundle/blob/master/Resources/doc/index.md)
before proceeding. It will help you get a grasp on how the bundle works, and why we use it.

## Installing VichUploaderBundle

Install the bundle with the help of composer:

```bash
docker-compose exec php composer require vich/uploader-bundle
```

This will create a new configuration file that you will need to slightly change
to make it look like this.

```yaml
# config/packages/vich_uploader.yaml
vich_uploader:
    db_driver: orm

    mappings:
        media_object:
            uri_prefix: /media
            upload_destination: '%kernel.project_dir%/public/media'
```

## Configuring the Entity Receiving the Uploaded File

In our exemple, we will create a `MediaObject` API resource. We will post files
to this resource endpoint, and then link the newly created resource to another
resource (in our case: Book).

The `MediaObject` resource is implemented like this:

```php
<?php
// src/Entity/MediaObject.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiProperty;
use ApiPlatform\Core\Annotation\ApiResource;
use App\Controller\CreateMediaObjectAction;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\HttpFoundation\File\File;
use Symfony\Component\Validator\Constraints as Assert;
use Vich\UploaderBundle\Mapping\Annotation as Vich;

/**
 * @ORM\Entity
 * @ApiResource(iri="http://schema.org/MediaObject", collectionOperations={
 *     "get",
 *     "post"={
 *         "method"="POST",
 *         "path"="/media_objects",
 *         "controller"=CreateMediaObjectAction::class,
 *         "defaults"={"_api_receive"=false},
 *     },
 * })
 * @Vich\Uploadable
 */
class MediaObject
{
    // ...

    /**
     * @var File|null
     * @Assert\NotNull()
     * @Vich\UploadableField(mapping="media_object", fileNameProperty="contentUrl")
     */
    public $file;

    /**
     * @var string|null
     * @ORM\Column(nullable=true)
     * @ApiProperty(iri="http://schema.org/contentUrl")
     */
    public $contentUrl;
}
```

## Handling File Upload

At this point, the entity is configured, but we still need to write the action
that handles the file upload.

```php
<?php
// src/Controller/CreateMediaObjectAction.php

namespace App\Controller;

use ApiPlatform\Core\Bridge\Symfony\Validator\Exception\ValidationException;
use App\Entity\MediaObject;
use App\Form\MediaObjectType;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\IsGranted;
use Symfony\Bridge\Doctrine\RegistryInterface;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Form\FormFactoryInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Serializer\SerializerInterface;
use Symfony\Component\Validator\Validator\ValidatorInterface;

final class CreateMediaObjectAction
{
    private $validator;
    private $doctrine;
    private $factory;

    public function __construct(RegistryInterface $doctrine, FormFactoryInterface $factory, ValidatorInterface $validator)
    {
        $this->validator = $validator;
        $this->doctrine = $doctrine;
        $this->factory = $factory;
    }

    /**
     * @IsGranted("ROLE_USER")
     */
    public function __invoke(Request $request): MediaObject
    {
        $mediaObject = new MediaObject();

        $form = $this->factory->create(MediaObjectType::class, $mediaObject);
        $form->handleRequest($request);
        if ($form->isSubmitted() && $form->isValid()) {
            $em = $this->doctrine->getManager();
            $em->persist($mediaObject);
            $em->flush();

            // Prevent the serialization of the file property
            $mediaObject->file = null;

            return $mediaObject;
        }

        // This will be handled by API Platform and returns a validation error.
        throw new ValidationException($this->validator->validate($mediaObject));
    }
}
```

As you can see, the action uses a form. You will need this form to be like this:

```php
<?php
// src/Form/MediaObjectType.php

namespace App\Form;

use App\Entity\MediaObject;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\FileType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

final class MediaObjectType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder
            // Configure each fields you want to be submitted here, like a classic form.
            ->add('file', FileType::class, [
                'label' => 'label.file',
                'required' => false,
            ])
        ;
    }

    public function configureOptions(OptionsResolver $resolver)
    {
        $resolver->setDefaults([
            'data_class' => MediaObject::class,
            'csrf_protection' => false,
        ]);
    }

    public function getBlockPrefix()
    {
        return '';
    }
}
```

## Making a Request to the `/media_objects` Endpoint

Your `/media_objects` endpoint is now ready to receive a `POST` request with a
file. This endpoint accepts standard `multipart/form-data` encoded data, but
not JSON data. You will need to format your request accordingly. After posting
your data, you will get a response looking like this:

```json
{
    "@type": "http://schema.org/ImageObject",
    "@id": "/media_objects/<id>",
    "contentUrl": "<filename>",
}
```

## Linking a MediaObject Resource to Another Resource

We now need to update our `Book` resource, so that we can link a `MediaObject`
to serve as the book cover.

We first need to edit our Book resource, and add a new property called `image`.

```php
<?php
// src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\HttpFoundation\File\File;
use Vich\UploaderBundle\Mapping\Annotation as Vich;

/**
 * @ORM\Entity
 * @ApiResource(iri="http://schema.org/Book")
 */
class Book
{
    // ...
    /**
     * @var MediaObject|null
     * @ORM\ManyToOne(targetEntity="App\Entity\MediaObject")
     * @ORM\JoinColumn(nullable=true)
     * @ApiProperty(iri="http://schema.org/image")
     */
    public $image;
}
```

By sending a POST request to create a new book, linked with the previously
uploaded cover, you can have a nice illustrated book record!

`POST /books`

```json
{
  "name": "The name",
  "image": "/media_objects/<id>"
}
```

Voila! You can now send files to your API, and link them to any other resources
in your app.
