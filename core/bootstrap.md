# Bootstrapping the Core Library

You may want to run a minimal version of API Platform without the full Symfony or Laravel framework.
This guide shows how to bootstrap the core library using only Symfony components and the Symfony
HttpKernel.

> [!NOTE]
>
> The
> [ApiPlatformProvider](https://github.com/api-platform/core/blob/main/src/Laravel/ApiPlatformProvider.php)
> registers all API Platform services for the Laravel framework. It can be a useful reference for
> understanding how services are wired together.

## Required Packages

```console
composer require \
    api-platform/metadata \
    api-platform/serializer \
    api-platform/state \
    api-platform/jsonld \
    api-platform/hydra \
    api-platform/symfony \
    symfony/http-kernel \
    symfony/event-dispatcher \
    symfony/routing \
    symfony/serializer \
    symfony/property-info \
    symfony/property-access \
    phpdocumentor/reflection-docblock \
    willdurand/negotiation
```

You may also want to install additional packages depending on your needs:

```console
composer require \
    symfony/validator           # validation support
```

## Full Bootstrap Example

Create the following file structure:

```text
├── bootstrap.php
├── composer.json
└── src/
    ├── Book.php
    ├── BookProcessor.php
    └── BookProvider.php
```

Create `src/Book.php`:

```php
<?php

namespace App;

use ApiPlatform\Metadata\ApiResource;

#[ApiResource(provider: BookProvider::class, processor: BookProcessor::class)]
class Book
{
    public int $id;
    public string $title = '';
}
```

Create `src/BookProvider.php`:

```php
<?php

namespace App;

use ApiPlatform\Metadata\CollectionOperationInterface;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;

class BookProvider implements ProviderInterface
{
    public function provide(
        Operation $operation,
        array $uriVariables = [],
        array $context = [],
    ): object|array|null {
        if ($operation instanceof CollectionOperationInterface) {
            $book = new Book();
            $book->id = 1;
            $book->title = 'API Platform';

            return [$book];
        }

        $book = new Book();
        $book->id = $uriVariables['id'];
        $book->title = 'API Platform';

        return $book;
    }
}
```

Create `src/BookProcessor.php`:

```php
<?php

namespace App;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;

class BookProcessor implements ProcessorInterface
{
    public function process(
        mixed $data,
        Operation $operation,
        array $uriVariables = [],
        array $context = [],
    ): mixed {
        // Persist your data here
        return $data;
    }
}
```

Then create `bootstrap.php`:

```php
<?php

require './vendor/autoload.php';

use App\BookProcessor;
use App\BookProvider;
use ApiPlatform\Hydra\Serializer\CollectionFiltersNormalizer;
use ApiPlatform\Hydra\Serializer\CollectionNormalizer as HydraCollectionNormalizer;
use ApiPlatform\Hydra\Serializer\PartialCollectionViewNormalizer;
use ApiPlatform\JsonLd\Action\ContextAction;
use ApiPlatform\JsonLd\ContextBuilder as JsonLdContextBuilder;
use ApiPlatform\JsonLd\Serializer\ItemNormalizer as JsonLdItemNormalizer;
use ApiPlatform\JsonLd\Serializer\ObjectNormalizer as JsonLdObjectNormalizer;
use ApiPlatform\Metadata\HttpOperation;
use ApiPlatform\Metadata\IdentifiersExtractor;
use ApiPlatform\Metadata\Operation\UnderscorePathSegmentNameGenerator;
use ApiPlatform\Metadata\Property\Factory\AttributePropertyMetadataFactory;
use ApiPlatform\Metadata\Property\Factory\PropertyInfoPropertyMetadataFactory;
use ApiPlatform\Metadata\Property\Factory\PropertyInfoPropertyNameCollectionFactory;
use ApiPlatform\Metadata\Property\Factory\SerializerPropertyMetadataFactory;
use ApiPlatform\Metadata\Resource\Factory\AlternateUriResourceMetadataCollectionFactory;
use ApiPlatform\Metadata\Resource\Factory\AttributesResourceMetadataCollectionFactory;
use ApiPlatform\Metadata\Resource\Factory\AttributesResourceNameCollectionFactory;
use ApiPlatform\Metadata\Resource\Factory\FiltersResourceMetadataCollectionFactory;
use ApiPlatform\Metadata\Resource\Factory\FormatsResourceMetadataCollectionFactory;
use ApiPlatform\Metadata\Resource\Factory\InputOutputResourceMetadataCollectionFactory;
use ApiPlatform\Metadata\Resource\Factory\LinkFactory;
use ApiPlatform\Metadata\Resource\Factory\LinkResourceMetadataCollectionFactory;
use ApiPlatform\Metadata\Resource\Factory\NotExposedOperationResourceMetadataCollectionFactory;
use ApiPlatform\Metadata\Resource\Factory\OperationNameResourceMetadataCollectionFactory;
use ApiPlatform\Metadata\Resource\Factory\PhpDocResourceMetadataCollectionFactory;
use ApiPlatform\Metadata\Resource\Factory\ResourceMetadataCollectionFactoryInterface;
use ApiPlatform\Metadata\Resource\Factory\ResourceNameCollectionFactoryInterface;
use ApiPlatform\Metadata\Resource\Factory\UriTemplateResourceMetadataCollectionFactory;
use ApiPlatform\Metadata\ResourceClassResolver;
use ApiPlatform\Metadata\UriVariablesConverter;
use ApiPlatform\Metadata\UriVariableTransformer\DateTimeUriVariableTransformer;
use ApiPlatform\Metadata\UriVariableTransformer\IntegerUriVariableTransformer;
use ApiPlatform\Metadata\UrlGeneratorInterface as ApiUrlGeneratorInterface;
use ApiPlatform\Serializer\ItemNormalizer;
use ApiPlatform\Serializer\JsonEncoder as ApiJsonLdEncoder;
use ApiPlatform\Serializer\Mapping\Factory\ClassMetadataFactory as ApiClassMetadataFactory;
use ApiPlatform\Serializer\SerializerContextBuilder;
use ApiPlatform\State\CallableProcessor;
use ApiPlatform\State\CallableProvider;
use ApiPlatform\State\ProcessorInterface;
use ApiPlatform\State\Provider\ContentNegotiationProvider;
use ApiPlatform\State\Provider\ReadProvider;
use ApiPlatform\State\Processor\RespondProcessor;
use ApiPlatform\State\Processor\SerializeProcessor;
use ApiPlatform\State\Processor\WriteProcessor;
use ApiPlatform\State\ProviderInterface;
use ApiPlatform\Symfony\Action\NotExposedAction;
use ApiPlatform\Symfony\Action\PlaceholderAction;
use ApiPlatform\Symfony\EventListener\AddFormatListener;
use ApiPlatform\Symfony\EventListener\ReadListener;
use ApiPlatform\Symfony\EventListener\RespondListener;
use ApiPlatform\Symfony\EventListener\SerializeListener;
use ApiPlatform\Symfony\EventListener\WriteListener;
use ApiPlatform\Symfony\Routing\IriConverter;
use ApiPlatform\Symfony\Routing\SkolemIriConverter;
use Negotiation\Negotiator;
use Psr\Container\ContainerInterface;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\RequestStack;
use Symfony\Component\HttpKernel\Controller\ArgumentResolver;
use Symfony\Component\HttpKernel\Controller\ControllerResolver;
use Symfony\Component\HttpKernel\EventListener\RouterListener;
use Symfony\Component\HttpKernel\HttpKernel;
use Symfony\Component\HttpKernel\Log\Logger;
use Symfony\Component\PropertyAccess\PropertyAccess;
use Symfony\Component\PropertyInfo\Extractor\PhpDocExtractor;
use Symfony\Component\PropertyInfo\Extractor\ReflectionExtractor;
use Symfony\Component\PropertyInfo\PropertyInfoExtractor;
use Symfony\Component\Routing\Generator\UrlGenerator;
use Symfony\Component\Routing\Generator\UrlGeneratorInterface;
use Symfony\Component\Routing\Matcher\UrlMatcher;
use Symfony\Component\Routing\RequestContext;
use Symfony\Component\Routing\Route;
use Symfony\Component\Routing\RouteCollection;
use Symfony\Component\Routing\RouterInterface;
use Symfony\Component\Serializer\Encoder\JsonEncoder;
use Symfony\Component\Serializer\Mapping\Factory\ClassMetadataFactory;
use Symfony\Component\Serializer\Mapping\Loader\AttributeLoader;
use Symfony\Component\Serializer\NameConverter\MetadataAwareNameConverter;
use Symfony\Component\Serializer\Normalizer\ArrayDenormalizer;
use Symfony\Component\Serializer\Normalizer\DateTimeNormalizer;
use Symfony\Component\Serializer\Normalizer\ObjectNormalizer;
use Symfony\Component\Serializer\Normalizer\UnwrappingDenormalizer;
use Symfony\Component\Serializer\Serializer;

// ──────────────────────────────────────────────
// 1. Configuration
// ──────────────────────────────────────────────

$defaultContext = [];
$formats = ['jsonld' => ['application/ld+json']];
$patchFormats = ['json' => ['application/merge-patch+json']];
$errorFormats = [
    'jsonproblem' => ['application/problem+json'],
    'jsonld' => ['application/ld+json'],
];

$logger = new Logger();

// ──────────────────────────────────────────────
// 2. Property Info
// ──────────────────────────────────────────────

$phpDocExtractor = new PhpDocExtractor();
$reflectionExtractor = new ReflectionExtractor();
$propertyInfo = new PropertyInfoExtractor(
    [$reflectionExtractor],
    [$phpDocExtractor, $reflectionExtractor],
    [$phpDocExtractor],
    [$reflectionExtractor],
    [$reflectionExtractor]
);

// ──────────────────────────────────────────────
// 3. Serializer Class Metadata
// ──────────────────────────────────────────────

$classMetadataFactory = new ClassMetadataFactory(new AttributeLoader());
$nameConverter = new MetadataAwareNameConverter($classMetadataFactory);

// ──────────────────────────────────────────────
// 4. Resource and Property Metadata Factories
// ──────────────────────────────────────────────

// Point this to the directory containing your #[ApiResource] classes
$resourceNameCollectionFactory = new AttributesResourceNameCollectionFactory([__DIR__ . '/src/']);
$resourceClassResolver = new ResourceClassResolver($resourceNameCollectionFactory);

// Property metadata: Attribute -> PropertyInfo -> Serializer
$propertyNameCollectionFactory = new PropertyInfoPropertyNameCollectionFactory($propertyInfo);
$propertyMetadataFactory = new AttributePropertyMetadataFactory();
$propertyMetadataFactory = new PropertyInfoPropertyMetadataFactory($propertyInfo, $propertyMetadataFactory);
$propertyMetadataFactory = new SerializerPropertyMetadataFactory(
    new ApiClassMetadataFactory($classMetadataFactory),
    $propertyMetadataFactory,
    $resourceClassResolver,
);

$pathSegmentNameGenerator = new UnderscorePathSegmentNameGenerator();
$linkFactory = new LinkFactory(
    $propertyNameCollectionFactory,
    $propertyMetadataFactory,
    $resourceClassResolver,
);

// Resource metadata: decorator chain (innermost to outermost)
// Each factory adds or transforms metadata before passing to the next.
$resourceMetadataFactory = new AlternateUriResourceMetadataCollectionFactory(
    new FiltersResourceMetadataCollectionFactory(
        new FormatsResourceMetadataCollectionFactory(
            new InputOutputResourceMetadataCollectionFactory(
                new PhpDocResourceMetadataCollectionFactory(
                    new OperationNameResourceMetadataCollectionFactory(
                        new LinkResourceMetadataCollectionFactory(
                            $linkFactory,
                            new UriTemplateResourceMetadataCollectionFactory(
                                $linkFactory,
                                $pathSegmentNameGenerator,
                                new NotExposedOperationResourceMetadataCollectionFactory(
                                    $linkFactory,
                                    new AttributesResourceMetadataCollectionFactory(
                                        null,
                                        $logger,
                                        [],
                                        false,
                                    ),
                                ),
                            ),
                        ),
                    ),
                ),
            ),
            $formats,
            $patchFormats,
        ),
    ),
);

// ──────────────────────────────────────────────
// 5. PSR-11 Container for Filters
// ──────────────────────────────────────────────

$filterLocator = new class implements ContainerInterface {
    public function get(string $id): mixed
    {
        return null;
    }

    public function has(string $id): bool
    {
        return false;
    }
};

// ──────────────────────────────────────────────
// 6. State Provider and Processor Locators
// ──────────────────────────────────────────────

$providerLocator = new class implements ContainerInterface {
    /** @var array<string, ProviderInterface> */
    public array $providers = [];

    public function get(string $id): mixed
    {
        return $this->providers[$id];
    }

    public function has(string $id): bool
    {
        return isset($this->providers[$id]);
    }
};
$providerLocator->providers[\App\BookProvider::class] = new BookProvider();

$processorLocator = new class implements ContainerInterface {
    /** @var array<string, ProcessorInterface> */
    public array $processors = [];

    public function get(string $id): mixed
    {
        return $this->processors[$id];
    }

    public function has(string $id): bool
    {
        return isset($this->processors[$id]);
    }
};
$processorLocator->processors[\App\BookProcessor::class] = new BookProcessor();

$callableProvider = new CallableProvider($providerLocator);
$callableProcessor = new CallableProcessor($processorLocator);

// ──────────────────────────────────────────────
// 7. Route Building
// ──────────────────────────────────────────────

$propertyAccessor = PropertyAccess::createPropertyAccessor();
$identifiersExtractor = new IdentifiersExtractor(
    $resourceMetadataFactory,
    $resourceClassResolver,
    $propertyNameCollectionFactory,
    $propertyMetadataFactory,
    $propertyAccessor,
);

// Build Symfony routes from resource metadata
function buildRoutes(
    ResourceNameCollectionFactoryInterface $resourceNameCollectionFactory,
    ResourceMetadataCollectionFactoryInterface $resourceMetadataFactory,
): RouteCollection {
    $routeCollection = new RouteCollection();

    foreach ($resourceNameCollectionFactory->create() as $resourceClass) {
        foreach ($resourceMetadataFactory->create($resourceClass) as $resourceMetadata) {
            foreach ($resourceMetadata->getOperations() as $operationName => $operation) {
                if ($operation->getRouteName()) {
                    continue;
                }

                if (SkolemIriConverter::$skolemUriTemplate === $operation->getUriTemplate()) {
                    continue;
                }

                $path = ($operation->getRoutePrefix() ?? '') . $operation->getUriTemplate();

                foreach ($operation->getUriVariables() ?? [] as $parameterName => $link) {
                    if (!$expandedValue = $link->getExpandedValue()) {
                        continue;
                    }

                    $path = str_replace(
                        sprintf('{%s}', $parameterName),
                        $expandedValue,
                        $path,
                    );
                }

                $route = new Route(
                    $path,
                    [
                        '_controller' => $operation->getController() ?? PlaceholderAction::class,
                        '_format' => null,
                        '_stateless' => $operation->getStateless(),
                        '_api_resource_class' => $resourceClass,
                        '_api_operation_name' => $operationName,
                    ] + ($operation->getDefaults() ?? []),
                    $operation->getRequirements() ?? [],
                    $operation->getOptions() ?? [],
                    $operation->getHost() ?? '',
                    $operation->getSchemes() ?? [],
                    [$operation->getMethod() ?? HttpOperation::METHOD_GET],
                    $operation->getCondition() ?? '',
                );

                $routeCollection->add($operationName, $route);
            }
        }
    }

    return $routeCollection;
}

$routes = buildRoutes($resourceNameCollectionFactory, $resourceMetadataFactory);

// Add the .well-known/genid route for blank nodes
$notExposedAction = new NotExposedAction();
$routes->add('api_genid', new Route(
    '/.well-known/genid/{id}',
    ['_controller' => $notExposedAction, '_format' => 'text', '_api_respond' => true],
));

// ──────────────────────────────────────────────
// 8. Router and URL Generator
// ──────────────────────────────────────────────

$requestContext = new RequestContext();
$matcher = new UrlMatcher($routes, $requestContext);
$generator = new UrlGenerator($routes, $requestContext);

// Adapter implementing Symfony's RouterInterface
class Router implements RouterInterface
{
    public function __construct(
        private RouteCollection $routes,
        private UrlMatcher $matcher,
        private UrlGeneratorInterface $generator,
        private RequestContext $context,
    ) {
    }

    public function getRouteCollection(): RouteCollection
    {
        return $this->routes;
    }

    public function match(string $pathinfo): array
    {
        return $this->matcher->match($pathinfo);
    }

    public function setContext(RequestContext $context): void
    {
        $this->context = $context;
    }

    public function getContext(): RequestContext
    {
        return $this->context;
    }

    public function generate(
        string $name,
        array $parameters = [],
        int $referenceType = self::ABSOLUTE_PATH,
    ): string {
        return $this->generator->generate($name, $parameters, $referenceType);
    }
}

// Adapter for API Platform's UrlGeneratorInterface
class ApiUrlGenerator implements ApiUrlGeneratorInterface
{
    public function __construct(private UrlGeneratorInterface $generator)
    {
    }

    public function generate(
        string $name,
        array $parameters = [],
        int $referenceType = self::ABS_PATH,
    ): string {
        return $this->generator->generate($name, $parameters, $referenceType ?: self::ABS_PATH);
    }
}

$router = new Router($routes, $matcher, $generator, $requestContext);
$apiUrlGenerator = new ApiUrlGenerator($generator);

// ──────────────────────────────────────────────
// 9. IRI Converter
// ──────────────────────────────────────────────

$uriVariablesConverter = new UriVariablesConverter(
    $propertyMetadataFactory,
    $resourceMetadataFactory,
    [new IntegerUriVariableTransformer(), new DateTimeUriVariableTransformer()],
);

$iriConverter = new IriConverter(
    $callableProvider,
    $router,
    $identifiersExtractor,
    $resourceClassResolver,
    $resourceMetadataFactory,
    $uriVariablesConverter,
    new SkolemIriConverter($router),
);

// ──────────────────────────────────────────────
// 10. Serializer
// ──────────────────────────────────────────────

$serializerContextBuilder = new SerializerContextBuilder($resourceMetadataFactory);
$objectNormalizer = new ObjectNormalizer();

// JSON-LD context builder
$jsonLdContextBuilder = new JsonLdContextBuilder(
    $resourceNameCollectionFactory,
    $resourceMetadataFactory,
    $propertyNameCollectionFactory,
    $propertyMetadataFactory,
    $apiUrlGenerator,
    $iriConverter,
    $nameConverter,
);

// Add the JSON-LD context route (required for @context URLs in responses)
$contextAction = new ContextAction(
    $jsonLdContextBuilder,
    $resourceNameCollectionFactory,
    $resourceMetadataFactory,
);
$routes->add('api_jsonld_context', new Route(
    '/contexts/{shortName}.{_format}',
    ['_controller' => $contextAction, '_format' => 'jsonld', '_api_respond' => true],
    ['shortName' => '.+'],
));

// JSON-LD normalizers
$jsonLdItemNormalizer = new JsonLdItemNormalizer(
    $resourceMetadataFactory,
    $propertyNameCollectionFactory,
    $propertyMetadataFactory,
    $iriConverter,
    $resourceClassResolver,
    $jsonLdContextBuilder,
    $propertyAccessor,
    $nameConverter,
    $classMetadataFactory,
    $defaultContext,
    null, // resourceAccessChecker
);

$jsonLdObjectNormalizer = new JsonLdObjectNormalizer(
    $objectNormalizer,
    $iriConverter,
    $jsonLdContextBuilder,
);

// Hydra normalizers (for collections and API metadata)
$hydraCollectionNormalizer = new HydraCollectionNormalizer(
    $jsonLdContextBuilder,
    $resourceClassResolver,
    $iriConverter,
    $defaultContext,
);

$hydraPartialCollectionNormalizer = new PartialCollectionViewNormalizer(
    $hydraCollectionNormalizer,
    'page',
    'pagination',
    $resourceMetadataFactory,
    $propertyAccessor,
);

$hydraCollectionFiltersNormalizer = new CollectionFiltersNormalizer(
    $hydraPartialCollectionNormalizer,
    $resourceMetadataFactory,
    $resourceClassResolver,
    $filterLocator,
);

// Core normalizer
$itemNormalizer = new ItemNormalizer(
    $propertyNameCollectionFactory,
    $propertyMetadataFactory,
    $iriConverter,
    $resourceClassResolver,
    $propertyAccessor,
    $nameConverter,
    $classMetadataFactory,
    $logger,
    $resourceMetadataFactory,
    null, // resourceAccessChecker
    $defaultContext,
);

// Symfony normalizers
$unwrappingDenormalizer = new UnwrappingDenormalizer($propertyAccessor);
$arrayDenormalizer = new ArrayDenormalizer();
$dateTimeNormalizer = new DateTimeNormalizer($defaultContext);

// Register normalizers with priorities (same as Symfony bundle)
$list = new \SplPriorityQueue();
$list->insert($unwrappingDenormalizer, 1000);
$list->insert($hydraCollectionFiltersNormalizer, -800);
$list->insert($jsonLdItemNormalizer, -890);
$list->insert($jsonLdObjectNormalizer, -995);
$list->insert($arrayDenormalizer, -990);
$list->insert($dateTimeNormalizer, -910);
$list->insert($itemNormalizer, -895);
$list->insert($objectNormalizer, -1000);

$encoders = [new JsonEncoder(), new ApiJsonLdEncoder('jsonld', new JsonEncoder())];
$serializer = new Serializer(iterator_to_array($list), $encoders);

// ──────────────────────────────────────────────
// 11. State Providers and Processors
// ──────────────────────────────────────────────

// Content negotiation provider (standalone: only negotiates format, does not read data)
$contentNegotiationProvider = new ContentNegotiationProvider(
    null,
    new Negotiator(),
    $formats,
    $errorFormats,
);

// Read provider: fetches data from the user's provider
$readProvider = new ReadProvider($callableProvider, $serializerContextBuilder, $logger);

// Processor chain: writes data, serializes, and creates the HTTP response
$respondProcessor = new RespondProcessor(
    $iriConverter,
    $resourceClassResolver,
    null,
    $resourceMetadataFactory,
);
$writeProcessor = new WriteProcessor(null, $callableProcessor);
$serializeProcessor = new SerializeProcessor(
    null,
    $serializer,
    $serializerContextBuilder,
);

// ──────────────────────────────────────────────
// 12. Event Listeners and HttpKernel
// ──────────────────────────────────────────────
// Note: this section is specific to the Symfony HttpKernel.
// When using Laravel, request handling is managed differently
// (see ApiPlatformProvider).

$formatListener = new AddFormatListener($contentNegotiationProvider, $resourceMetadataFactory);
$readListener = new ReadListener(
    $readProvider,
    $resourceMetadataFactory,
    $uriVariablesConverter,
);
$writeListener = new WriteListener(
    $writeProcessor,
    $resourceMetadataFactory,
    $uriVariablesConverter,
);
$serializeListener = new SerializeListener($serializeProcessor, $resourceMetadataFactory);
$respondListener = new RespondListener($respondProcessor, $resourceMetadataFactory);

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new RouterListener($matcher, new RequestStack()));
$dispatcher->addListener('kernel.request', [$formatListener, 'onKernelRequest'], 28);
$dispatcher->addListener('kernel.request', [$readListener, 'onKernelRequest'], 4);
$dispatcher->addListener('kernel.view', [$writeListener, 'onKernelView'], 32);
$dispatcher->addListener('kernel.view', [$serializeListener, 'onKernelView'], 16);
$dispatcher->addListener('kernel.view', [$respondListener, 'onKernelView'], 8);

$kernel = new HttpKernel(
    $dispatcher,
    new ControllerResolver(),
    new RequestStack(),
    new ArgumentResolver(),
);

$request = Request::createFromGlobals();
$response = $kernel->handle($request);
$response->send();
$kernel->terminate($request, $response);
```

## Running

Make sure your `composer.json` includes the PSR-4 autoload for the `src/` directory:

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
```

Then run `composer dump-autoload` and start the PHP built-in server:

```console
php -S localhost:8000 bootstrap.php
```

Then access your API:

```console
curl http://localhost:8000/books
curl http://localhost:8000/books/1
```

## Going Further

This bootstrap provides a minimal JSON-LD/Hydra API. To add more features, you can:

<!-- prettier-ignore -->
- **Deserialization**: add `DeserializeProvider` and `DeserializeListener` for POST/PUT/PATCH
- **Validation**: add `ValidateProvider` and `ValidateListener` with `symfony/validator`
- **HAL/JSON:API**: register the corresponding normalizers with `api-platform/hal` or `api-platform/jsonapi`
- **OpenAPI**: add `OpenApiFactory` and `OpenApiNormalizer` for API documentation

Refer to the
[ApiPlatformProvider](https://github.com/api-platform/core/blob/main/src/Laravel/ApiPlatformProvider.php)
for a complete example of all available services and their wiring.
