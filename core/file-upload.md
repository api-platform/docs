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
 * @ApiResource(
 *     iri="http://schema.org/MediaObject",
 *     normalizationContext={
 *         "groups"={"media_object_read"}
 *     },
 *     collectionOperations={
 *         "post"={
 *             "controller"=CreateMediaObjectAction::class,
 *             "deserialize"=false,
 *             "security"="is_granted('ROLE_USER')",
 *             "validation_groups"={"Default", "media_object_create"},
 *             "openapi_context"={
 *                 "requestBody"={
 *                     "content"={
 *                         "multipart/form-data"={
 *                             "schema"={
 *                                 "type"="object",
 *                                 "properties"={
 *                                     "file"={
 *                                         "type"="string",
 *                                         "format"="binary"
 *                                     }
 *                                 }
 *                             }
 *                         }
 *                     }
 *                 }
 *             }
 *         },
 *         "get"
 *     },
 *     itemOperations={
 *         "get"
 *     }
 * )
 * @Vich\Uploadable
 */
class MediaObject
{
    /**
     * @var int|null
     *
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue
     * @ORM\Id
     */
    protected $id;

    /**
     * @var string|null
     *
     * @ApiProperty(iri="http://schema.org/contentUrl")
     * @Groups({"media_object_read"})
     */
    public $contentUrl;

    /**
     * @var File|null
     *
     * @Assert\NotNull(groups={"media_object_create"})
     * @Vich\UploadableField(mapping="media_object", fileNameProperty="filePath")
     */
    public $file;

    /**
     * @var string|null
     *
     * @ORM\Column(nullable=true)
     */
    public $filePath;

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

A [normalizer](serialization.md#normalization) could be used to set the `contentUrl` property:

```php
<?php
// api/src/Serializer/MediaObjectNormalizer.php

namespace App\Serializer;

use App\Entity\MediaObject;
use Symfony\Component\Serializer\Normalizer\ContextAwareNormalizerInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerAwareInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerAwareTrait;
use Vich\UploaderBundle\Storage\StorageInterface;

final class MediaObjectNormalizer implements ContextAwareNormalizerInterface, NormalizerAwareInterface
{
    use NormalizerAwareTrait;

    private const ALREADY_CALLED = 'MEDIA_OBJECT_NORMALIZER_ALREADY_CALLED';

    public function __construct(private StorageInterface $storage)
    {
    }

    public function normalize($object, ?string $format = null, array $context = []): array|string|int|float|bool|\ArrayObject|null
    {
        $context[self::ALREADY_CALLED] = true;

        $object->contentUrl = $this->storage->resolveUri($object, 'file');

        return $this->normalizer->normalize($object, $format, $context);
    }

    public function supportsNormalization($data, ?string $format = null, array $context = []): bool
    {
        if (isset($context[self::ALREADY_CALLED])) {
            return false;
        }

        return $data instanceof MediaObject;
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
  "@type": "http://schema.org/MediaObject",
  "@id": "/media_objects/<id>",
  "contentUrl": "<url>"
}
```

## Accessing Your Media Objects Directly

You will need to modify your Caddyfile to allow the above `contentUrl` to be accessed directly. If you followed the above configuration for the VichUploaderBundle, that will be in `api/public/media`. Add your folder to the list of path matches, e.g. `|^/media/|`:
```caddyfile
...
# Matches requests for HTML documents, for static files and for Next.js files,
# except for known API paths and paths with extensions handled by API Platform
@pwa expression `(
        {header.Accept}.matches("\\btext/html\\b")
        && !{path}.matches("(?i)(?:^/docs|^/graphql|^/bundles/|^/media/|^/_profiler|^/_wdt|\\.(?:json|html$|csv$|ya?ml$|xml$))")
...
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
 * @ApiResource(iri="http://schema.org/Book")
 */
class Book
{
    // ...

    /**
     * @var MediaObject|null
     *
     * @ORM\ManyToOne(targetEntity=MediaObject::class)
     * @ORM\JoinColumn(nullable=true)
     * @ApiProperty(iri="http://schema.org/image")
     */
    public $image;
    
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

## Testing

To test your upload with `ApiTestCase`, you can write a method as below:

```
<?php
// tests/MediaObjectTest.php

namespace App\Tests;

use ApiPlatform\Core\Bridge\Symfony\Bundle\Test\ApiTestCase;
use App\Entity\MediaObject;
use Hautelook\AliceBundle\PhpUnit\RefreshDatabaseTrait;
use Symfony\Component\HttpFoundation\File\UploadedFile;

class MediaObjectTest extends ApiTestCase
{
    use RefreshDatabaseTrait;

    public function testCreateAMediaObject(): void
    {
        $file = new UploadedFile('fixtures/files/image.png', 'image.png');
        $client = self::createClient();

        $client->request('POST', '/media_objects', [
            'headers' => ['Content-Type' => 'multipart/form-data'],
            'extra' => [
                // If you have additionnal fields in your MediaObject entity, use the parameters
                'parameters' => [
                    'title' => 'My file uploaded'
                ],
                'files' => [
                    'file' => $file
                ]
            ]
        ]);
        $this->assertResponseIsSuccessful();
        $this->assertMatchesResourceItemJsonSchema(MediaObject::class);
        $this->assertJsonContains([
            'title' => 'My file uploaded',
        ]);
    }
}
```
