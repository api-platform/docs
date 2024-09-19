# Bootstraping the core library

You may want to run a minimal version of API Platform. This one file runs API Platform (without GraphQL, Eloquent, Doctrine MongoDB...).
It requires the following Composer packages:

```console
composer require \
    api-platform/core \
    phpdocumentor/reflection-docblock \
    symfony/property-info \
    symfony/routing \
    symfony/validator
```

The minimal version of API Platform:

```php
<?php

require './vendor/autoload.php';

use ApiPlatform\Action\EntrypointAction;
use ApiPlatform\Action\ExceptionAction;
use ApiPlatform\Action\NotExposedAction;
use ApiPlatform\Action\PlaceholderAction;
use ApiPlatform\Api\IdentifiersExtractor;
use ApiPlatform\Api\ResourceClassResolver;
use ApiPlatform\Api\UriVariablesConverter;
use ApiPlatform\Api\UriVariableTransformer\DateTimeUriVariableTransformer;
use ApiPlatform\Api\UriVariableTransformer\IntegerUriVariableTransformer;
use ApiPlatform\Api\UrlGeneratorInterface as ApiUrlGeneratorInterface;
use ApiPlatform\Symfony\Validator\EventListener\ValidationExceptionListener;
use ApiPlatform\Documentation\DocumentationInterface;
use ApiPlatform\Symfony\EventListener\AddFormatListener;
use ApiPlatform\Symfony\EventListener\DeserializeListener;
use ApiPlatform\Symfony\EventListener\ExceptionListener;
use ApiPlatform\Symfony\EventListener\ReadListener;
use ApiPlatform\Symfony\EventListener\WriteListener;
use ApiPlatform\State\Pagination\PaginationOptions;
use ApiPlatform\Symfony\EventListener\RespondListener;
use ApiPlatform\Symfony\EventListener\SerializeListener;
use ApiPlatform\Hal\Serializer\CollectionNormalizer as HalCollectionNormalizer;
use ApiPlatform\Hal\Serializer\EntrypointNormalizer as HalEntrypointNormalizer;
use ApiPlatform\Hal\Serializer\ItemNormalizer as HalItemNormalizer;
use ApiPlatform\Hal\Serializer\ObjectNormalizer as HalObjectNormalizer;
use ApiPlatform\Hydra\EventListener\AddLinkHeaderListener;
use ApiPlatform\Hydra\Serializer\CollectionFiltersNormalizer;
use ApiPlatform\Hydra\Serializer\CollectionNormalizer as HydraCollectionNormalizer;
use ApiPlatform\Hydra\Serializer\ConstraintViolationListNormalizer as HydraConstraintViolationListNormalizer;
use ApiPlatform\Hydra\Serializer\DocumentationNormalizer as HydraDocumentationNormalizer;
use ApiPlatform\Hydra\Serializer\EntrypointNormalizer as HydraEntrypointNormalizer;
use ApiPlatform\Hydra\Serializer\ErrorNormalizer as HydraErrorNormalizer;
use ApiPlatform\Hydra\Serializer\PartialCollectionViewNormalizer;
use ApiPlatform\JsonLd\Action\ContextAction;
use ApiPlatform\JsonLd\Serializer\ItemNormalizer as JsonLdItemNormalizer;
use ApiPlatform\JsonLd\Serializer\ObjectNormalizer as JsonLdObjectNormalizer;
use ApiPlatform\JsonLd\ContextBuilder as JsonLdContextBuilder;
use ApiPlatform\JsonSchema\SchemaFactory;
use ApiPlatform\JsonSchema\TypeFactory;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\CollectionOperationInterface;
use ApiPlatform\Metadata\HttpOperation;
use ApiPlatform\Metadata\Operation;
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
use ApiPlatform\OpenApi\Factory\OpenApiFactory;
use ApiPlatform\OpenApi\Factory\OpenApiFactoryInterface;
use ApiPlatform\OpenApi\Options as OpenApiOptions;
use ApiPlatform\OpenApi\Serializer\OpenApiNormalizer;
use ApiPlatform\Operation\UnderscorePathSegmentNameGenerator;
use ApiPlatform\Problem\Serializer\ConstraintViolationListNormalizer as ProblemConstraintViolationListNormalizer;
use ApiPlatform\Problem\Serializer\ErrorNormalizer;
use ApiPlatform\Serializer\ItemNormalizer;
use ApiPlatform\Serializer\JsonEncoder as JsonLdEncoder;
use ApiPlatform\Serializer\Mapping\Factory\ClassMetadataFactory as ApiClassMetadataFactory;
use ApiPlatform\Serializer\SerializerContextBuilder;
use ApiPlatform\State\CallableProcessor;
use ApiPlatform\State\CallableProvider;
use ApiPlatform\State\ProcessorInterface;
use ApiPlatform\State\ProviderInterface;
use ApiPlatform\Symfony\Routing\IriConverter;
use ApiPlatform\Symfony\EventListener\ValidateListener;
use ApiPlatform\Symfony\Messenger\Metadata\MessengerResourceMetadataCollectionFactory;
use ApiPlatform\Symfony\Routing\SkolemIriConverter;
use ApiPlatform\Validator\ValidatorInterface;
use Negotiation\Negotiator;
use Psr\Container\ContainerInterface;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\RequestStack;
use Symfony\Component\HttpKernel\Log\Logger;
use Symfony\Component\HttpKernel\Controller\ArgumentResolver;
use Symfony\Component\HttpKernel\Controller\ControllerResolver;
use Symfony\Component\HttpKernel\EventListener\ErrorListener;
use Symfony\Component\HttpKernel\EventListener\RouterListener;
use Symfony\Component\HttpKernel\HttpKernel;
use Symfony\Component\PropertyAccess\PropertyAccess;
use Symfony\Component\Routing\Matcher\UrlMatcher;
use Symfony\Component\Routing\RequestContext;
use Symfony\Component\Routing\Route;
use Symfony\Component\Routing\RouteCollection;
use Symfony\Component\PropertyInfo\Extractor\PhpDocExtractor;
use Symfony\Component\PropertyInfo\Extractor\ReflectionExtractor;
use Symfony\Component\PropertyInfo\PropertyInfoExtractor;
use Symfony\Component\Routing\Generator\UrlGenerator;
use Symfony\Component\Routing\Generator\UrlGeneratorInterface;
use Symfony\Component\Routing\Matcher\UrlMatcherInterface;
use Symfony\Component\Routing\RouterInterface;
use Symfony\Component\Serializer\Mapping\Factory\ClassMetadataFactory;
use Symfony\Component\Serializer\Mapping\Loader\AnnotationLoader;
use Symfony\Component\Serializer\NameConverter\MetadataAwareNameConverter;
use Symfony\Component\Validator\Validation;
use Symfony\Component\Serializer\Encoder\JsonEncoder;
use Symfony\Component\Serializer\Normalizer\ArrayDenormalizer;
use Symfony\Component\Serializer\Normalizer\ConstraintViolationListNormalizer;
use Symfony\Component\Serializer\Normalizer\DataUriNormalizer;
use Symfony\Component\Serializer\Normalizer\DateIntervalNormalizer;
use Symfony\Component\Serializer\Normalizer\DateTimeNormalizer;
use Symfony\Component\Serializer\Normalizer\DateTimeZoneNormalizer;
use Symfony\Component\Serializer\Normalizer\JsonSerializableNormalizer;
use Symfony\Component\Serializer\Normalizer\ObjectNormalizer;
use Symfony\Component\Serializer\Normalizer\ProblemNormalizer;
use Symfony\Component\Serializer\Normalizer\UnwrappingDenormalizer;
use Symfony\Component\Serializer\Serializer;

$debug = true;
$defaultContext = [];
$patchFormats = ['json' => ['application/merge-patch+json'], 'jsonapi' => ['application/vnd.api+json']];
$formats = ['jsonld' => ['application/ld+json']];
$errorFormats = [
    'jsonproblem' => ['application/problem+json'],
    'jsonld' => ['application/ld+json'],
    'jsonapi' => ['application/vnd.api+json']
];

$configuration = [
    'collection' => [
        'pagination' => [
            'page_parameter_name' => 'page',
            'enabled_parameter_name' => 'pagination'
        ]
    ]
];

$exceptionToStatus = [
    # The 4 following handlers are registered by default, keep those lines to prevent unexpected side effects
    \Symfony\Component\Serializer\Exception\ExceptionInterface::class => 400,
    \ApiPlatform\Exception\InvalidArgumentException::class => 400,
    \ApiPlatform\Exception\FilterValidationException::class => 400,
    \Doctrine\ORM\OptimisticLockException::class => 409,
];

$logger = new Logger();

$phpDocExtractor = new PhpDocExtractor();
$reflectionExtractor = new ReflectionExtractor();

$propertyInfo = new PropertyInfoExtractor(
    [$reflectionExtractor],
    [$phpDocExtractor, $reflectionExtractor],
    [$phpDocExtractor],
    [$reflectionExtractor],
    [$reflectionExtractor]
);


$classMetadataFactory = new ClassMetadataFactory(new AnnotationLoader());

final class FilterLocator implements ContainerInterface 
{
    private $filters = [];
    public function get(string $id) {
        return $this->filters[$id] ?? null;
    }

    public function has(string $id): bool {
        return isset($this->filter[$id]); 
    }
}

$filterLocator = new FilterLocator();
$pathSegmentNameGenerator = new UnderscorePathSegmentNameGenerator(); 

$resourceNameCollectionFactory = new AttributesResourceNameCollectionFactory([__DIR__.'/../src/']);
$resourceClassResolver = new ResourceClassResolver($resourceNameCollectionFactory);
$propertyMetadataFactory = new PropertyInfoPropertyMetadataFactory($propertyInfo);
$propertyMetadataFactory = new SerializerPropertyMetadataFactory(new ApiClassMetadataFactory($classMetadataFactory), $propertyMetadataFactory, $resourceClassResolver);

$propertyNameCollectionFactory = new PropertyInfoPropertyNameCollectionFactory($propertyInfo);
$linkFactory = new LinkFactory(
    $propertyNameCollectionFactory,
    $propertyMetadataFactory,
    $resourceClassResolver
);

// CachedResourceMetadataCollectionFactory decoration-priority="-10"
// MessengerResourceMetadataCollectionFactory decoration-priority="50"
// AlternateUriResourceMetadataCollectionFactory decoration-priority="200"
// FiltersResourceMetadataCollectionFactory decoration-priority="200"
// FormatsResourceMetadataCollectionFactory decoration-priority="200"
// InputOutputResourceMetadataCollectionFactory decoration-priority="200"
// PhpDocResourceMetadataCollectionFactory decoration-priority="200"
// OperationNameResourceMetadataCollectionFactory decoration-priority="200"
// LinkResourceMetadataCollectionFactory decoration-priority="500"
// UriTemplateResourceMetadataCollectionFactory decoration-priority="500"
// NotExposedOperationResourceMetadataCollectionFactory decoration-priority="700"
// ExtractorResourceMetadataCollectionFactory  decoration-priority="800"

// AttributesResourceMetadataCollectionFactory decorated

$resourceMetadataFactory = new MessengerResourceMetadataCollectionFactory(
    new AlternateUriResourceMetadataCollectionFactory(
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
                                        new AttributesResourceMetadataCollectionFactory(null, $logger, [], false)
                                    )
                                )
                            )
                        )
                    )
                ),
                $formats,
                $patchFormats,
            )
        )
    )
);

$providerCollection = new class implements ContainerInterface {
    public array $providers = [];
    public function get($id) {
        return $this->providers[$id];
    }

    public function has($id): bool {
        return isset($this->providers[$id]);
    }
};
$stateProviders = new CallableProvider($providerCollection);

$processorCollection = new class implements ContainerInterface {
    public array $processors = [];
    public function get($id) {
        return $this->processors[$id];
    }

    public function has($id): bool {
        return isset($this->processors['id']);
    }
};
$stateProcessors = new CallableProcessor($processorCollection);

class Validator implements ValidatorInterface {
    private $validator;
    public function __construct($validator) 
    {
        $this->validator = $validator;
    }

    public function validate(object $data, array $context = []): void {
        $this->validator->validate($data, $context);
    }
}

$validator = new Validator(Validation::createValidator());
$validateListener = new ValidateListener($validator, $resourceMetadataFactory);

#[ApiResource(provider: \BookProvider::class)]
class Book
{
    public int $id;
}

class BookProvider implements ProviderInterface
{
    public function provide(Operation $operation, array $uriVariables = [], array $context = []): object|array|null
    {
        $book = new Book();
        if ($operation instanceof CollectionOperationInterface) {
            return [$book];
        }

        $book->id = $uriVariables['id'];
        return $book;
    }
}

$dataProvider = new BookProvider();
$providerCollection->providers[BookProvider::class] = $dataProvider;

class BookProcessor implements ProcessorInterface {
    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []) {}
}

$bookProcessor = new BookProcessor();
$processorCollection->processors[BookProcessor::class] = $bookProcessor;

$propertyAccessor = PropertyAccess::createPropertyAccessor();
$identifiersExtractor = new IdentifiersExtractor($resourceMetadataFactory, $resourceClassResolver, $propertyNameCollectionFactory, $propertyMetadataFactory, $propertyAccessor);

class ApiLoader {
    private $resourceNameCollectionFactory;
    private $resourceMetadataFactory;

    public function __construct(
        ResourceNameCollectionFactoryInterface $resourceNameCollectionFactory, 
        ResourceMetadataCollectionFactoryInterface $resourceMetadataFactory
    ) {
        $this->resourceNameCollectionFactory = $resourceNameCollectionFactory;
        $this->resourceMetadataFactory = $resourceMetadataFactory;
    }

    public function load(): RouteCollection
    {
        $routeCollection = new RouteCollection();

        foreach ($this->resourceNameCollectionFactory->create() as $resourceClass) {
            foreach ($this->resourceMetadataFactory->create($resourceClass) as $resourceMetadata) {
                foreach ($resourceMetadata->getOperations() as $operationName => $operation) {
                    if ($operation->getRouteName()) {
                        continue;
                    }

                    if (SkolemIriConverter::$skolemUriTemplate === $operation->getUriTemplate()) {
                        continue;
                    }

                    $path = ($operation->getRoutePrefix() ?? '').$operation->getUriTemplate();
                    foreach ($operation->getUriVariables() ?? [] as $parameterName => $link) {
                        if (!$expandedValue = $link->getExpandedValue()) {
                            continue;
                        }

                        $path = str_replace(sprintf('{%s}', $parameterName), $expandedValue, $path);
                    }

                    if (($controller = $operation->getController()) && !$this->container->has($controller)) {
                        throw new RuntimeException(sprintf('There is no builtin action for the "%s" operation. You need to define the controller yourself.', $operationName));
                    }

                    $route = new Route(
                        $path,
                        [
                            '_controller' => $controller ?? PlaceholderAction::class,
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
                        $operation->getCondition() ?? ''
                    );

                    $routeCollection->add($operationName, $route);
                }
            }
        }

        return $routeCollection;
    }
}

$apiLoader = new ApiLoader($resourceNameCollectionFactory, $resourceMetadataFactory);
$routes = $apiLoader->load();

$requestContext = new RequestContext();
$matcher = new UrlMatcher($routes, $requestContext);
$generator = new UrlGenerator($routes, $requestContext);

class Router implements RouterInterface 
{
    private $routes;
    private $context;
    private $matcher;
    private $generator;

    public function __construct(RouteCollection $routes, UrlMatcherInterface $matcher, UrlGeneratorInterface $generator, RequestContext $requestContext) 
    {
        $this->routes = $routes;
        $this->matcher = $matcher;
        $this->generator = $generator;
        $this->context = $requestContext;
    }

    public function getRouteCollection() {
        return $this->routes;
    }

    public function match(string $pathinfo): array {
        return $this->matcher->match($pathinfo);
    }

    public function setContext(RequestContext $context) {
        $this->context = $context;
    }

    public function getContext(): RequestContext {
        return $this->context;
    }

    public function generate(string $name, array $parameters = [], int $referenceType = self::ABSOLUTE_PATH): string {
        return $this->generator->generate($name, $parameters, $referenceType);
    }
}

class ApiUrlGenerator implements ApiUrlGeneratorInterface {
    private $generator;

    public function __construct(UrlGeneratorInterface $generator)
    {
        $this->generator = $generator;
    }

    public function generate($name, $parameters = [], $referenceType = self::ABS_PATH): string {
        return $this->generator->generate($name, $parameters, $referenceType ?: self::ABS_PATH);
    }
}

$apiUrlGenerator = new ApiUrlGenerator($generator);

$router = new Router($routes, $matcher, $generator, $requestContext);

$uriVariableTransformers = [
    new IntegerUriVariableTransformer(),
    new DateTimeUriVariableTransformer(),
];

$iriConverter = new IriConverter(
    $stateProviders, 
    $router, 
    $identifiersExtractor, 
    $resourceClassResolver,
    $resourceMetadataFactory,
    new UriVariablesConverter($propertyMetadataFactory, $resourceMetadataFactory, $uriVariableTransformers),
    new SkolemIriConverter($router)
);

$writeListener = new WriteListener(
    $stateProcessors,
    $iriConverter, 
    $resourceClassResolver, 
    $resourceMetadataFactory, 
    /**new UriVariablesConverter($propertyMetadataFactory, $resourceMetadataFactory, $uriVariableTransformers)*/ null,
);

$serializerContextBuilder = new SerializerContextBuilder($resourceMetadataFactory);

$objectNormalizer = new ObjectNormalizer();

$nameConverter = new MetadataAwareNameConverter($classMetadataFactory);
$jsonLdContextBuilder = new JsonLdContextBuilder($resourceNameCollectionFactory, $resourceMetadataFactory, $propertyNameCollectionFactory, $propertyMetadataFactory, $apiUrlGenerator, $iriConverter, $nameConverter);
$jsonLdItemNormalizer = new JsonLdItemNormalizer($resourceMetadataFactory, $propertyNameCollectionFactory, $propertyMetadataFactory, $iriConverter, $resourceClassResolver, $jsonLdContextBuilder, $propertyAccessor, $nameConverter, $classMetadataFactory, $defaultContext, /** resource access checker **/ null);
$jsonLdObjectNormalizer = new JsonLdObjectNormalizer($objectNormalizer, $iriConverter, $jsonLdContextBuilder);
$jsonLdEncoder = new JsonLdEncoder('jsonld', new JsonEncoder());

$problemConstraintViolationListNormalizer = new ProblemConstraintViolationListNormalizer([], $nameConverter, $defaultContext);

$hydraCollectionNormalizer = new HydraCollectionNormalizer($jsonLdContextBuilder, $resourceClassResolver, $iriConverter, $resourceMetadataFactory, $defaultContext);
$hydraPartialCollectionNormalizer = new PartialCollectionViewNormalizer($hydraCollectionNormalizer, $configuration['collection']['pagination']['page_parameter_name'], $configuration['collection']['pagination']['enabled_parameter_name'], $resourceMetadataFactory, $propertyAccessor);
$hydraCollectionFiltersNormalizer = new CollectionFiltersNormalizer($hydraPartialCollectionNormalizer, $resourceMetadataFactory, $resourceClassResolver, $filterLocator);
$hydraErrorNormalizer = new HydraErrorNormalizer($apiUrlGenerator, $debug, $defaultContext);
$hydraEntrypointNormalizer = new HydraEntrypointNormalizer($resourceMetadataFactory, $iriConverter, $apiUrlGenerator);
$hydraDocumentationNormalizer = new HydraDocumentationNormalizer($resourceMetadataFactory, $propertyNameCollectionFactory, $propertyMetadataFactory, $resourceClassResolver, $apiUrlGenerator, $nameConverter);
$hydraConstraintViolationNormalizer = new HydraConstraintViolationListNormalizer($apiUrlGenerator, [], $nameConverter);

$problemErrorNormalizer = new ErrorNormalizer($debug, $defaultContext);

// $expressionLanguage = new ExpressionLanguage();
// $resourceAccessChecker = new ResourceAccessChecker(
//      $expressionLanguage,
// );

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
    /**$resourceAccessChecker **/ null,
    $defaultContext
);

$arrayDenormalizer = new ArrayDenormalizer();
$problemNormalizer = new ProblemNormalizer($debug, $defaultContext);
$jsonserializableNormalizer = new JsonSerializableNormalizer($classMetadataFactory, $nameConverter, $defaultContext);
$dateTimeNormalizer = new DateTimeNormalizer($defaultContext);
$dataUriNormalizer = new DataUriNormalizer();
$dateIntervalNormalizer = new DateIntervalNormalizer($defaultContext);
$dateTimeZoneNormalizer = new DateTimeZoneNormalizer();
$constraintViolationListNormalizer = new ConstraintViolationListNormalizer($defaultContext, $nameConverter);
$unwrappingDenormalizer = new UnwrappingDenormalizer($propertyAccessor);

$halItemNormalizer = new HalItemNormalizer($propertyNameCollectionFactory, $propertyMetadataFactory, $iriConverter, $resourceClassResolver, $propertyAccessor, $nameConverter, $classMetadataFactory, $defaultContext, $resourceMetadataFactory, /** resourceAccessChecker **/ null);
$halItemNormalizer = new HalItemNormalizer($propertyNameCollectionFactory, $propertyMetadataFactory, $iriConverter, $resourceClassResolver, $propertyAccessor, $nameConverter, $classMetadataFactory, $defaultContext, $resourceMetadataFactory, /** resourceAccessChecker **/ null);

$halEntrypointNormalizer = new HalEntrypointNormalizer($resourceMetadataFactory, $iriConverter, $apiUrlGenerator);
$halCollectionNormalizer = new HalCollectionNormalizer($resourceClassResolver, $configuration['collection']['pagination']['page_parameter_name'], $resourceMetadataFactory);
$halObjectNormalizer = new HalObjectNormalizer($objectNormalizer, $iriConverter);

$openApiNormalizer = new OpenApiNormalizer($objectNormalizer);

$list = new \SplPriorityQueue();
$list->insert($unwrappingDenormalizer, 1000);
$list->insert($halItemNormalizer, -890);
$list->insert($hydraConstraintViolationNormalizer, -780);
$list->insert($hydraEntrypointNormalizer, -800);
$list->insert($hydraErrorNormalizer, -800);
$list->insert($hydraCollectionFiltersNormalizer, -800);
$list->insert($halEntrypointNormalizer, -800);
$list->insert($halCollectionNormalizer, -985);
$list->insert($halObjectNormalizer, -995);
$list->insert($jsonLdItemNormalizer, -890);
$list->insert($problemConstraintViolationListNormalizer, -780);
$list->insert($problemErrorNormalizer, -810);
$list->insert($jsonLdObjectNormalizer, -995);
$list->insert($constraintViolationListNormalizer, -915);
$list->insert($arrayDenormalizer, -990);
$list->insert($dateTimeZoneNormalizer, -915);
$list->insert($dateIntervalNormalizer, -915);
$list->insert($dataUriNormalizer, -920);
$list->insert($dateTimeNormalizer, -910);
$list->insert($jsonserializableNormalizer, -900);
$list->insert($problemNormalizer, -890);
$list->insert($objectNormalizer, -1000);
$list->insert($itemNormalizer, -895);
// $list->insert($uuidDenormalizer, -895); //Todo ramsey uuid support ?
$list->insert($openApiNormalizer, -780);

// TODO: JSON-API support
/**
 * api_platform.jsonapi.normalizer.error                       -790       ApiPlatform\JsonApi\Serializer\ErrorNormalizer
 * api_platform.jsonapi.normalizer.constraint_violation_list   -780       ApiPlatform\JsonApi\Serializer\ConstraintViolationListNormalizer
 * api_platform.openapi.normalizer.api_gateway                 -780       ApiPlatform\Swagger\Serializer\ApiGatewayNormalizer
 * api_platform.jsonapi.normalizer.entrypoint                  -800       ApiPlatform\JsonApi\Serializer\EntrypointNormalizer
 * api_platform.jsonapi.normalizer.collection                  -985       ApiPlatform\JsonApi\Serializer\CollectionNormalizer
 * api_platform.jsonapi.normalizer.item                        -890       ApiPlatform\JsonApi\Serializer\ItemNormalizer
 * api_platform.jsonapi.normalizer.object                      -995       ApiPlatform\JsonApi\Serializer\ObjectNormalizer
 */

$encoders = [new JsonEncoder(), $jsonLdEncoder];
$serializer = new Serializer(iterator_to_array($list), $encoders);

$serializeListener = new SerializeListener($serializer, $serializerContextBuilder, $resourceMetadataFactory);
$respondListener = new RespondListener($resourceMetadataFactory);
$formatListener = new AddFormatListener(new Negotiator(), $resourceMetadataFactory, $formats);
$readListener = new ReadListener($stateProviders, $resourceMetadataFactory, $serializerContextBuilder);
$deserializeListener = new DeserializeListener($serializer, $serializerContextBuilder, $resourceMetadataFactory);
$addLinkHeaderListener = new AddLinkHeaderListener($apiUrlGenerator);
$validationExceptionListener = new ValidationExceptionListener($serializer, $errorFormats, $exceptionToStatus);

$controller = new ExceptionAction($serializer, $errorFormats, $exceptionToStatus);
$errorListener = new ErrorListener($controller);
$exceptionListener = new ExceptionListener($errorListener);

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new RouterListener($matcher, new RequestStack()));
$dispatcher->addListener('kernel.view', [$validateListener, 'onKernelView'], 64);
$dispatcher->addListener('kernel.view', [$writeListener, 'onKernelView'], 32);
$dispatcher->addListener('kernel.view', [$serializeListener, 'onKernelView'], 16);
// TODO: ApiPlatform\EventListener\QueryParameterValidateListener, prio 16   
$dispatcher->addListener('kernel.view', [$respondListener, 'onKernelView'], 8);
$dispatcher->addListener('kernel.request', [$formatListener, 'onKernelRequest'], 28);
$dispatcher->addListener('kernel.request', [$readListener, 'onKernelRequest'], 4);
$dispatcher->addListener('kernel.request', [$deserializeListener, 'onKernelRequest'], 2);
$dispatcher->addListener('kernel.exception', [$validationExceptionListener, 'onKernelException'], 2);
// $dispatcher->addListener('kernel.exception', [$exceptionListener, 'onKernelException'], -96);
$dispatcher->addListener('kernel.response', [$addLinkHeaderListener, 'onKernelResponse'], 2);

/*
 * TODO: 
 * api_platform.security.listener.request.deny_access     kernel.request      onSecurity                  3          ApiPlatform\Security\EventListener\DenyAccessListener
 *   "                                                    kernel.request      onSecurityPostDenormalize   1                                                                   
 * api_platform.swagger.listener.ui                       kernel.request      onKernelRequest                        ApiPlatform\Bridge\Symfony\Bundle\EventListener\SwaggerUiListener
 * api_platform.http_cache.listener.response.configure    kernel.response     onKernelResponse            -1         ApiPlatform\HttpCache\EventListener\AddHeadersListener
*/

final class DocumentationAction 
{
    private $openApiFactory;
    public function __construct(OpenApiFactoryInterface $openApiFactory)
    {
        $this->openApiFactory = $openApiFactory;
    }

    public function __invoke(Request $request): DocumentationInterface
    {
        $context = ['base_url' => $request->getBaseUrl(), 'spec_version' => 3];
        if ($request->query->getBoolean('api_gateway')) {
            $context['api_gateway'] = true;
        }

        return $this->openApiFactory->__invoke($context);
    }
}

$paginationOptions = new PaginationOptions();
$openApiOptions = new OpenApiOptions('API Platform');
$jsonSchemaTypeFactory = new TypeFactory($resourceClassResolver);
$jsonSchemaFactory = new SchemaFactory($jsonSchemaTypeFactory, $resourceMetadataFactory, $propertyNameCollectionFactory, $propertyMetadataFactory, $nameConverter, $resourceClassResolver);

$openApiFactory = new OpenApiFactory($resourceNameCollectionFactory, $resourceMetadataFactory, $propertyNameCollectionFactory, $propertyMetadataFactory, $jsonSchemaFactory, $jsonSchemaTypeFactory, $filterLocator);
$documentationAction = new DocumentationAction($openApiFactory);
$routes->add('api_doc', new Route('/docs.{_format}', ['_controller' => $documentationAction, '_format' => null, '_api_respond' => true]));

$entryPointAction = new EntrypointAction($resourceNameCollectionFactory);
$routes->add('api_entrypoint', new Route('/{index}.{_format}', ['_controller' => $entryPointAction, '_format' => null, '_api_respond' => true, 'index' => 'index'], ['index' => 'index']));

$contextAction = new ContextAction($jsonLdContextBuilder, $resourceNameCollectionFactory, $resourceMetadataFactory);
$routes->add('api_jsonld_context', new Route('/contexts/{shortName}.{_format}', ['_controller' => $contextAction, '_format' => 'jsonld', '_api_respond' => true], ['shortName' => '.+']));

$notExposedAction = new NotExposedAction();
$routes->add('api_genid', new Route('/.well-known/genid/{id}', ['_controller' => $notExposedAction, '_format' => 'text', '_api_respond' => true]));

$controllerResolver = new ControllerResolver();
$argumentResolver = new ArgumentResolver();

$kernel = new HttpKernel($dispatcher, $controllerResolver, new RequestStack(), $argumentResolver);

$request = Request::createFromGlobals();
$response = $kernel->handle($request);
$response->send();
$kernel->terminate($request, $response);
```
