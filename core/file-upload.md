# Handling File Upload

As common a problem as it may seem, handling file upload requires a custom
implementation in your app. This page will guide you in handling file upload in
your API, with the help of
[VichUploaderBundle](https://github.com/dustin10/VichUploaderBundle). It is
recommended you [read the documentation of
VichUploaderBundle](https://github.com/dustin10/VichUploaderBundle/blob/master/docs/index.md)
before proceeding. It will help you get a grasp on how the bundle works, and why we use it.

## Installing VichUploaderBundle

Install the bundle with the help of Composer:

```console
docker-compose exec php \
    composer require vich/uploader-bundle
```

This will create a new configuration file that you will need to slightly change
to make it look like this.

```yaml
# api/config/packages/vich_uploader.yaml
vich_uploader:
    db_driver: orm

    mappings:
        media_object:
            uri_prefix: /media
            upload_destination: '%kernel.project_dir%/public/media'
            # Will rename uploaded files using a uniqueid as a prefix.
            namer: Vich\UploaderBundle\Naming\OrignameNamer
```

## Configuring the Entity Receiving the Uploaded File

In our example, we will create a `MediaObject` API resource. We will post files
to this resource endpoint, and then link the newly created resource to another
resource (in our case: Book).

The `MediaObject` resource is implemented like this:

```php
<?php
// api/src/Entity/MediaObject.php
namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiProperty;
use ApiPlatform\Core\Annotation\ApiResource;
use App\Controller\CreateMediaObjectAction;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\HttpFoundation\File\File;
use Symfony\Component\Serializer\Annotation\Groups;
use Symfony\Component\Validator\Constraints as Assert;
use Vich\UploaderBundle\Mapping\Annotation as Vich;

/**
 * @ORM\Entity
 * @Vich\Uploadable
 */
#[ApiResource(iri: "https://schema.org/MediaObject",
      normalizationContext: [
          "groups" => ["media_object_read"]
      ],
      collectionOperations: [
          "post" => [
              "controller" => CreateMediaObjectAction::class,
              "deserialize" => false,
              "security" => "is_granted('ROLE_USER')",
              "validation_groups" => ["Default", "media_object_create"],
              "openapi_context" => [
                  "requestBody" => [
                      "content" => [
                          "multipart/form-data" => [
                              "schema" => [
                                  "type" => "object",
                                  "properties" => [
                                      "file" => [
                                          "type" => "string",
                                          "format" => "binary"
                                      ]
                                  ]
                              ]
                          ]
                      ]
                  ]
              ]
          ],
          "get"
      ],
      itemOperations: [
          "get"
      ]
)]
class MediaObject
{
    /**
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue
     * @ORM\Id
     */
    protected ?int $id;

    #[ApiProperty(iri: "https://schema.org/contentUrl")]
    #[Groups(["media_object_read"])]
    public ?string $contentUrl;

    /**
     * @Vich\UploadableField(mapping="media_object", fileNameProperty="filePath")
     */
    #[Assert\NotNull(groups: ["media_object_create"])]
    public ?File $file;

    /**
     * @ORM\Column(nullable=true)
     */
    public ?string $filePath;

    public function getId(): ?int
    {
        return $this->id;
    }
}
```

## The Controller

At this point, the entity is configured, but we still need to write the action
that handles the file upload.

```php
<?php
// api/src/Controller/CreateMediaObjectAction.php

namespace App\Controller;

use App\Entity\MediaObject;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Exception\BadRequestHttpException;

final class CreateMediaObjectAction
{
    public function __invoke(Request $request): MediaObject
    {
        $uploadedFile = $request->files->get('file');
        if (!$uploadedFile) {
            throw new BadRequestHttpException('"file" is required');
        }

        $mediaObject = new MediaObject();
        $mediaObject->file = $uploadedFile;

        return $mediaObject;
    }
}
```

## Resolving the File URL

Returning the plain file path on the filesystem where the file is stored is not useful for the client, which needs a
URL to work with.

An [event subscriber](events.md#custom-event-listeners) could be used to set the `contentUrl` property:

```php
<?php
// api/src/EventSubscriber/ResolveMediaObjectContentUrlSubscriber.php

namespace App\EventSubscriber;

use ApiPlatform\Core\EventListener\EventPriorities;
use ApiPlatform\Core\Util\RequestAttributesExtractor;
use App\Entity\MediaObject;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Event\ViewEvent;
use Symfony\Component\HttpKernel\KernelEvents;
use Vich\UploaderBundle\Storage\StorageInterface;

final class ResolveMediaObjectContentUrlSubscriber implements EventSubscriberInterface
{
    private $storage;

    public function __construct(StorageInterface $storage)
    {
        $this->storage = $storage;
    }

    public static function getSubscribedEvents(): array
    {
        return [
            KernelEvents::VIEW => ['onPreSerialize', EventPriorities::PRE_SERIALIZE],
        ];
    }

    public function onPreSerialize(ViewEvent $event): void
    {
        $controllerResult = $event->getControllerResult();
        $request = $event->getRequest();

        if ($controllerResult instanceof Response || !$request->attributes->getBoolean('_api_respond', true)) {
            return;
        }

        if (!($attributes = RequestAttributesExtractor::extractAttributes($request)) || !\is_a($attributes['resource_class'], MediaObject::class, true)) {
            return;
        }

        $mediaObjects = $controllerResult;

        if (!is_iterable($mediaObjects)) {
            $mediaObjects = [$mediaObjects];
        }

        foreach ($mediaObjects as $mediaObject) {
            if (!$mediaObject instanceof MediaObject) {
                continue;
            }

            $mediaObject->contentUrl = $this->storage->resolveUri($mediaObject, 'file');
        }
    }
}
```

## Making a Request to the `/media_objects` Endpoint

Your `/media_objects` endpoint is now ready to receive a `POST` request with a
file. This endpoint accepts standard `multipart/form-data`-encoded data, but
not JSON data. You will need to format your request accordingly. After posting
your data, you will get a response looking like this:

```json
{
  "@type": "https://schema.org/MediaObject",
  "@id": "/media_objects/<id>",
  "contentUrl": "<url>"
}
```

## Linking a MediaObject Resource to Another Resource

We now need to update our `Book` resource, so that we can link a `MediaObject`
to serve as the book cover.

We first need to edit our Book resource, and add a new property called `image`.

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Annotation\ApiProperty;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\HttpFoundation\File\File;
use Vich\UploaderBundle\Mapping\Annotation as Vich;

/**
 * @ORM\Entity
 */
#[ApiResource(iri: "https://schema.org/Book")]
class Book
{
    // ...

    /**
     * @ORM\ManyToOne(targetEntity=MediaObject::class)
     * @ORM\JoinColumn(nullable=true)
     */
    #[ApiResource(iri: "https://schema.org/image")]
    public ?MediaObject $image;
    
    // ...
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

Voilà! You can now send files to your API, and link them to any other resource
in your app.
