# Handling File Upload

As common a problem as it may seem, handling file upload requires a custom
implementation in your app. This page will guide you in handling file upload in
your API, with the help of
[VichUploaderBundle](https://github.com/dustin10/VichUploaderBundle). It is
recommended you [read the documentation of
VichUploaderBundle](https://github.com/dustin10/VichUploaderBundle/blob/master/docs/index.md)
before proceeding. It will help you get a grasp on how the bundle works, and why we use it.

**Note**: Uploading files won't work in `PUT` or `PATCH` requests, you must use `POST` method to upload files.
See [the related issue on Symfony](https://github.com/symfony/symfony/issues/9226) and [the related bug in PHP](https://bugs.php.net/bug.php?id=55815) talking about this behavior.

Enable the multipart format globally in order to use it as the input format of your resource: 

```yaml
api_platform:
    formats:
        multipart: ['multipart/form-data']
        jsonld: ['application/ld+json']
```

## Installing VichUploaderBundle

Install the bundle with the help of Composer:

```console
docker compose exec php \
    composer require vich/uploader-bundle
```

This will create a new configuration file that you will need to slightly change
to make it look like this.

```yaml
# api/config/packages/vich_uploader.yaml

vich_uploader:
    db_driver: orm
    metadata:
        type: attribute
    mappings:
        media_object:
            uri_prefix: /media
            upload_destination: '%kernel.project_dir%/public/media'
            # Will rename uploaded files using a uniqueid as a suffix.
            namer: Vich\UploaderBundle\Naming\SmartUniqueNamer
```

## Uploading to a Dedicated Resource

In this example, we will create a `MediaObject` API resource. We will post files
to this resource endpoint, and then link the newly created resource to another
resource (in our case: `Book`).

### Configuring the Resource Receiving the Uploaded File

The `MediaObject` resource is implemented like this:

```php
<?php
// api/src/Entity/MediaObject.php

namespace App\Entity;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\Post;
use ApiPlatform\OpenApi\Model;
use App\State\SaveMediaObject;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\HttpFoundation\File\File;
use Symfony\Component\Serializer\Annotation\Groups;
use Symfony\Component\Validator\Constraints as Assert;
use Vich\UploaderBundle\Mapping\Annotation as Vich;

#[Vich\Uploadable]
#[ORM\Entity]
#[ApiResource(
    normalizationContext: ['groups' => ['media_object:read']], 
    types: ['https://schema.org/MediaObject'],
    outputFormats: ['jsonld' => ['application/ld+json']],
    operations: [
        new Get(),
        new GetCollection(),
        new Post(
            inputFormats: ['multipart' => ['multipart/form-data']],
            openapi: new Model\Operation(
                requestBody: new Model\RequestBody(
                    content: new \ArrayObject([
                        'multipart/form-data' => [
                            'schema' => [
                                'type' => 'object', 
                                'properties' => [
                                    'file' => [
                                        'type' => 'string', 
                                        'format' => 'binary'
                                    ]
                                ]
                            ]
                        ]
                    ])
                )
            )
        )
    ]
)]
class MediaObject
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    #[ApiProperty(types: ['https://schema.org/contentUrl'], writable: false)]
    #[Groups(['media_object:read'])]
    public ?string $contentUrl = null;

    #[Vich\UploadableField(mapping: 'media_object', fileNameProperty: 'filePath')]
    #[Assert\NotNull]
    public ?File $file = null;

    #[ApiProperty(writable: false)]
    #[ORM\Column(nullable: true)] 
    public ?string $filePath = null;

    public function getId(): ?int
    {
        return $this->id;
    }
}
```
Note: From V3.3 onwards, `'multipart/form-data'` must either be including in the global API-Platform config, either in `formats` or `defaults->inputFormats`, or defined as an `inputFormats` parameter on an operation by operation basis.

### Resolving the File URL

Returning the plain file path on the filesystem where the file is stored is not useful for the client, which needs a
URL to work with.

A [normalizer](serialization.md#normalization) could be used to set the `contentUrl` property:

```php
<?php
// api/src/Serializer/MediaObjectNormalizer.php

namespace App\Serializer;

use App\Entity\MediaObject;
use Vich\UploaderBundle\Storage\StorageInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

class MediaObjectNormalizer implements NormalizerInterface
{

  private const ALREADY_CALLED = 'MEDIA_OBJECT_NORMALIZER_ALREADY_CALLED';

  public function __construct(
    #[Autowire(service: 'api_platform.jsonld.normalizer.item')]
    private readonly NormalizerInterface $normalizer,
    private readonly StorageInterface $storage
  ) {
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

  public function getSupportedTypes(?string $format): array
  {
    return [
      MediaObject::class => true,
    ];
  }
}

```

### Making a Request to the `/media_objects` Endpoint

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

### Accessing Your Media Objects Directly

You will need to modify your `Caddyfile` to allow the above `contentUrl` to be accessed directly. If you followed the above configuration for the VichUploaderBundle, that will be in `api/public/media`. Add your folder to the list of path matches, e.g. `|^/media/|`:

<!-- markdownlint-disable no-hard-tabs -->
```patch
	# Matches requests for HTML documents, for static files and for Next.js files,
	# except for known API paths and paths with extensions handled by API Platform
	@pwa expression `(
			header({'Accept': '*text/html*'})
			&& !path(
-				'/docs*', '/graphql*', '/bundles*', '/contexts*', '/_profiler*', '/_wdt*',
+				'/media*', '/docs*', '/graphql*', '/bundles*', '/contexts*', '/_profiler*', '/_wdt*',
				'*.json*', '*.html', '*.csv', '*.yml', '*.yaml', '*.xml'
			)
		)
		|| path('/favicon.ico', '/manifest.json', '/robots.txt', '/_next*', '/sitemap*')`
```
<!-- markdownlint-enable no-hard-tabs -->

### Linking a MediaObject Resource to Another Resource

We now need to update our `Book` resource, so that we can link a `MediaObject`
to serve as the book cover.

We first need to edit our Book resource, and add a new property called `image`.

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\ApiProperty;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\HttpFoundation\File\File;
use Vich\UploaderBundle\Mapping\Annotation as Vich;

#[ORM\Entity]
#[ApiResource(types: ['https://schema.org/Book'])]
class Book
{
    // ...

    #[ORM\ManyToOne(targetEntity: MediaObject::class)]
    #[ORM\JoinColumn(nullable: true)]
    #[ApiProperty(types: ['https://schema.org/image'])]
    public ?MediaObject $image = null;
    
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

### Testing

To test your upload with `ApiTestCase`, you can write a method as below:

```php
<?php
// tests/MediaObjectTest.php

namespace App\Tests;

use ApiPlatform\Symfony\Bundle\Test\ApiTestCase;
use App\Entity\MediaObject;
use Hautelook\AliceBundle\PhpUnit\RefreshDatabaseTrait;
use Symfony\Component\HttpFoundation\File\UploadedFile;

class MediaObjectTest extends ApiTestCase
{
    use RefreshDatabaseTrait;

    public function testCreateAMediaObject(): void
    {
        // The file "image.jpg" is the folder fixtures which is in the project dir
        $file = new UploadedFile(__DIR__ . '/../fixtures/image.jpg', 'image.jpg');
        $client = self::createClient();

        $client->request('POST', 'http://localhost:8888/api/media_objects', [
          'headers' => ['Content-Type' => 'multipart/form-data'],
          'extra' => [
            // If you have additional fields in your MediaObject entity, use the parameters.
            'parameters' => [
                // 'title' => 'title'
            ],
            'files' => [
              'file' => $file,
            ],
          ]
        ]);
        $this->assertResponseIsSuccessful();
        $this->assertMatchesResourceItemJsonSchema(MediaObject::class);
        $this->assertJsonContains([
            // 'title' => 'My file uploaded',
        ]);
    }
}
```

## Uploading to an Existing Resource with its Fields

In this example, the file will be included in an existing resource (in our case: `Book`).
The file and the resource fields will be posted to the resource endpoint.

This example will use a custom `multipart/form-data` decoder to deserialize the resource instead of a custom controller.

### Configuring the Existing Resource Receiving the Uploaded File

The `Book` resource needs to be modified like this:

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\Post;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\HttpFoundation\File\File;
use Symfony\Component\Serializer\Annotation\Groups;
use Vich\UploaderBundle\Mapping\Annotation as Vich;

/**
 * @Vich\Uploadable
 */
#[ORM\Entity]
#[ApiResource(
    normalizationContext: ['groups' => ['book:read']], 
    denormalizationContext: ['groups' => ['book:write']], 
    types: ['https://schema.org/Book'],
    operations: [
        new GetCollection(),
        new Post(
            outputFormats: ['jsonld' => ['application/ld+json']],
            inputFormats: ['multipart' => ['multipart/form-data']]
        )
    ]
)]
class Book
{
    // ...

    #[ApiProperty(types: ['https://schema.org/contentUrl'])]
    #[Groups(['book:read'])]
    public ?string $contentUrl = null;

    /**
     * @Vich\UploadableField(mapping="media_object", fileNameProperty="filePath")
     */
    #[Groups(['book:write'])]
    public ?File $file = null;

    #[ORM\Column(nullable: true)] 
    public ?string $filePath = null;
    
    // ...
}
```

### Handling the Multipart Deserialization

By default, Symfony is not able to decode `multipart/form-data`-encoded data.
We need to create our own decoder to do it:

```php
<?php
// api/src/Encoder/MultipartDecoder.php

namespace App\Encoder;

use Symfony\Component\HttpFoundation\RequestStack;
use Symfony\Component\Serializer\Encoder\DecoderInterface;

final class MultipartDecoder implements DecoderInterface
{
    public const FORMAT = 'multipart';

    public function __construct(private RequestStack $requestStack)
    {
    }

    public function decode(string $data, string $format, array $context = []): ?array
    {
        $request = $this->requestStack->getCurrentRequest();

        if (!$request) {
            return null;
        }

        return array_map(static function (string $element) {
            // Multipart form values will be encoded in JSON.
            $decoded = json_decode($element, true);

            return \is_array($decoded) ? $decoded : $element;
        }, $request->request->all()) + $request->files->all();
    }

    public function supportsDecoding(string $format): bool
    {
        return self::FORMAT === $format;
    }
}
```

If you're not using `autowiring` and `autoconfiguring`, don't forget to register the service and tag it as `serializer.encoder`.

We also need to make sure the field containing the uploaded file is not denormalized:

```php
<?php
// api/src/Serializer/UploadedFileDenormalizer.php

namespace App\Serializer;

use Symfony\Component\HttpFoundation\File\File;
use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;

final class UploadedFileDenormalizer implements DenormalizerInterface
{
    public function denormalize($data, string $type, string $format = null, array $context = []): File
    {
        return $data;
    }

    public function supportsDenormalization($data, $type, $format = null, array $context = []): bool
    {
        return $data instanceof File;
    }

    public function getSupportedTypes(?string $format): array
    {
        return [
            'object' => null,
            '*' => false,
            File::class => true,
        ];
    }
}
```

If you're not using `autowiring` and `autoconfiguring`, don't forget to register the service and tag it as `serializer.normalizer`.

For resolving the file URL, you can use a custom normalizer, like shown in [the previous example](#resolving-the-file-url).
