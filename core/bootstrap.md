# Bootstraping the core library

You may want to run a minimal version of API Platform. This one file runs API Platform (without graphql, doctrine and mongodb).
It requires the following composer packages:

```
"symfony/http-kernel": "^5.1",
"symfony/routing": "^5.1",
"symfony/event-dispatcher": "^5.1",
"api-platform/core": "^v2.6.0-alpha.1",
"doctrine/annotations": "^1.11",
"doctrine/common": "^3.0",
"symfony/property-info": "^5.1",
"phpdocumentor/reflection-docblock": "^5.2",
"symfony/validator": "^5.1",
"willdurand/negotiation": "^3.0.0"
```

The minimal version of API Platform:

```
<?php

require './vendor/autoload.php';

use ApiPlatform\Core\Action\EntrypointAction;
use ApiPlatform\Core\Action\ExceptionAction;
use ApiPlatform\Core\Action\PlaceholderAction;
use ApiPlatform\Core\Api\IdentifiersExtractor;
use ApiPlatform\Core\Api\IdentifiersExtractorInterface;
use ApiPlatform\Core\Api\OperationType;
use ApiPlatform\Core\Api\ResourceClassResolver;
use ApiPlatform\Core\Api\UrlGeneratorInterface as ApiUrlGeneratorInterface;
use ApiPlatform\Core\Bridge\Symfony\Routing\RouteNameGenerator;
use ApiPlatform\Core\Bridge\Symfony\Validator\EventListener\ValidationExceptionListener;
use ApiPlatform\Core\Documentation\DocumentationInterface;
use ApiPlatform\Core\EventListener\AddFormatListener;
use ApiPlatform\Core\EventListener\DeserializeListener;
use ApiPlatform\Core\EventListener\ExceptionListener;
use ApiPlatform\Core\EventListener\ReadListener;
use ApiPlatform\Core\EventListener\WriteListener;
use ApiPlatform\Core\Bridge\Symfony\PropertyInfo\Metadata\Property\PropertyInfoPropertyMetadataFactory;
use ApiPlatform\Core\Bridge\Symfony\PropertyInfo\Metadata\Property\PropertyInfoPropertyNameCollectionFactory;
use ApiPlatform\Core\Bridge\Symfony\Routing\IriConverter;
use ApiPlatform\Core\Bridge\Symfony\Routing\RouteNameResolver;
use ApiPlatform\Core\DataPersister\DataPersisterInterface;
use ApiPlatform\Core\DataProvider\ContextAwareCollectionDataProviderInterface;
use ApiPlatform\Core\DataProvider\DenormalizedIdentifiersAwareItemDataProviderInterface;
use ApiPlatform\Core\DataProvider\PaginationOptions;
use ApiPlatform\Core\DataProvider\RestrictedDataProviderInterface;
use ApiPlatform\Core\EventListener\RespondListener;
use ApiPlatform\Core\EventListener\SerializeListener;
use ApiPlatform\Core\Hal\Serializer\CollectionNormalizer as HalCollectionNormalizer;
use ApiPlatform\Core\Hal\Serializer\EntrypointNormalizer as HalEntrypointNormalizer;
use ApiPlatform\Core\Hal\Serializer\ItemNormalizer as HalItemNormalizer;
use ApiPlatform\Core\Hal\Serializer\ObjectNormalizer as HalObjectNormalizer;
use ApiPlatform\Core\Hydra\EventListener\AddLinkHeaderListener;
use ApiPlatform\Core\Hydra\Serializer\CollectionFiltersNormalizer;
use ApiPlatform\Core\Hydra\Serializer\CollectionNormalizer as HydraCollectionNormalizer;
use ApiPlatform\Core\Hydra\Serializer\ConstraintViolationListNormalizer as HydraConstraintViolationListNormalizer;
use ApiPlatform\Core\Hydra\Serializer\DocumentationNormalizer as HydraDocumentationNormalizer;
use ApiPlatform\Core\Hydra\Serializer\EntrypointNormalizer as HydraEntrypointNormalizer;
use ApiPlatform\Core\Hydra\Serializer\ErrorNormalizer as HydraErrorNormalizer;
use ApiPlatform\Core\Hydra\Serializer\PartialCollectionViewNormalizer;
use ApiPlatform\Core\Identifier\IdentifierConverter;
use ApiPlatform\Core\Identifier\Normalizer\IntegerDenormalizer;
use ApiPlatform\Core\JsonLd\Action\ContextAction;
use ApiPlatform\Core\JsonLd\Serializer\ItemNormalizer as JsonLdItemNormalizer;
use ApiPlatform\Core\JsonLd\Serializer\ObjectNormalizer as JsonLdObjectNormalizer;
use ApiPlatform\Core\JsonLd\ContextBuilder as JsonLdContextBuilder;
use ApiPlatform\Core\JsonSchema\SchemaFactory;
use ApiPlatform\Core\JsonSchema\TypeFactory;
use ApiPlatform\Core\Metadata\Property\Factory\AnnotationPropertyMetadataFactory;
use ApiPlatform\Core\Metadata\Property\Factory\InheritedPropertyMetadataFactory;
use ApiPlatform\Core\Metadata\Property\Factory\InheritedPropertyNameCollectionFactory;
use ApiPlatform\Core\Metadata\Property\Factory\SerializerPropertyMetadataFactory;
use ApiPlatform\Core\Metadata\Resource\Factory\AnnotationResourceFilterMetadataFactory;
use ApiPlatform\Core\Metadata\Resource\Factory\AnnotationResourceMetadataFactory;
use ApiPlatform\Core\Metadata\Resource\Factory\AnnotationResourceNameCollectionFactory;
use ApiPlatform\Core\Metadata\Resource\Factory\FormatsResourceMetadataFactory;
use ApiPlatform\Core\Metadata\Resource\Factory\InputOutputResourceMetadataFactory;
use ApiPlatform\Core\Metadata\Resource\Factory\OperationResourceMetadataFactory;
use ApiPlatform\Core\Metadata\Resource\Factory\ResourceMetadataFactoryInterface;
use ApiPlatform\Core\Metadata\Resource\Factory\ResourceNameCollectionFactoryInterface;
use ApiPlatform\Core\Metadata\Resource\Factory\ShortNameResourceMetadataFactory;
use ApiPlatform\Core\Metadata\Resource\ResourceMetadata;
use ApiPlatform\Core\OpenApi\Factory\OpenApiFactory;
use ApiPlatform\Core\OpenApi\Factory\OpenApiFactoryInterface;
use ApiPlatform\Core\OpenApi\Options as OpenApiOptions;
use ApiPlatform\Core\OpenApi\Serializer\OpenApiNormalizer;
use ApiPlatform\Core\Operation\Factory\SubresourceOperationFactory;
use ApiPlatform\Core\Operation\UnderscorePathSegmentNameGenerator;
use ApiPlatform\Core\PathResolver\OperationPathResolver;
use ApiPlatform\Core\PathResolver\OperationPathResolverInterface;
use ApiPlatform\Core\Problem\Serializer\ConstraintViolationListNormalizer as ProblemConstraintViolationListNormalizer;
use ApiPlatform\Core\Problem\Serializer\ErrorNormalizer;
use ApiPlatform\Core\Serializer\ItemNormalizer;
use ApiPlatform\Core\Serializer\JsonEncoder as JsonLdEncoder;
use ApiPlatform\Core\Serializer\SerializerContextBuilder;
use ApiPlatform\Core\Validator\EventListener\ValidateListener;
use ApiPlatform\Core\Validator\ValidatorInterface;
use Doctrine\Common\Annotations\AnnotationReader;
use Negotiation\Negotiator;
use Psr\Container\ContainerInterface;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\RequestStack;
use Symfony\Component\HttpFoundation\Response;
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
use Symfony\Component\Serializer\SerializerInterface;
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

$allowPlainIdentifiers = false;
$debug = true;
$defaultContext = [];
$dataTransformers = [];
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
    \ApiPlatform\Core\Exception\InvalidArgumentException::class => 400,
    \ApiPlatform\Core\Exception\FilterValidationException::class => 400,
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

$doctrineAnnotationReader = new AnnotationReader();
$classMetadataFactory = new ClassMetadataFactory(new AnnotationLoader($doctrineAnnotationReader));

final class FilterLocator implements ContainerInterface 
{
    private $filters = [];
    public function get($id) {
        return $this->filters[$id] ?? null;
    }

    public function has($id) {
        return isset($this->filter[$id]); 
    }
}

$apiPlatformAnnotationReader = $doctrineAnnotationReader;
if (\PHP_VERSION_ID >= 80000) {
    $apiPlatformAnnotationReader = null;
}

$filterLocator = new FilterLocator();

$resourceNameCollectionFactory = new AnnotationResourceNameCollectionFactory($apiPlatformAnnotationReader, ['./src/Entity']);
$resourceMetadataFactory = new FormatsResourceMetadataFactory(new OperationResourceMetadataFactory(new ShortNameResourceMetadataFactory(new InputOutputResourceMetadataFactory(new AnnotationResourceFilterMetadataFactory($apiPlatformAnnotationReader, new AnnotationResourceMetadataFactory($apiPlatformAnnotationReader, null)))), $patchFormats), $formats, $patchFormats);;
$propertyNameCollectionFactory = new InheritedPropertyNameCollectionFactory($resourceNameCollectionFactory, new PropertyInfoPropertyNameCollectionFactory($propertyInfo));
$resourceClassResolver = new ResourceClassResolver($resourceNameCollectionFactory);
$propertyMetadataFactory = new SerializerPropertyMetadataFactory($resourceMetadataFactory, $classMetadataFactory, new InheritedPropertyMetadataFactory($resourceNameCollectionFactory, new PropertyInfoPropertyMetadataFactory($propertyInfo, new AnnotationPropertyMetadataFactory($apiPlatformAnnotationReader))), $resourceClassResolver);

class Validator implements ValidatorInterface {
    private $validator;
    public function __construct($validator) 
    {
        $this->validator = $validator;
    }

    public function validate($data, array $context = []) {
        return $this->validator->validate($data, $context);
    }
}

$validator = new Validator(Validation::createValidator());
$validateListener = new ValidateListener($validator, $resourceMetadataFactory);

class DataProvider implements DenormalizedIdentifiersAwareItemDataProviderInterface, RestrictedDataProviderInterface, ContextAwareCollectionDataProviderInterface

{
    public function getCollection(string $resourceClass, string $operationName = null, array $context = [])
    {
        $book = new Book();
        $book->id = '1';
        return [$book];
    }

    public function getItem(string $resourceClass, $identifiers, string $operationName = null, array $context = []) {
        $book = new Book();
        $book->id = $identifiers['id'];
        return $book;
    }

    public function supports(string $resourceClass, string $operationName = null, array $context = []): bool {
        return true;
    }
}

$dataProvider = new DataProvider();

class DataPersister implements DataPersisterInterface {
    public function supports($data): bool
    {
        return true;
    }
    public function persist($data) {}
    public function remove($data) {}
}

$dataPersister = new DataPersister();

$propertyAccessor = PropertyAccess::createPropertyAccessor();
$identifiersExtractor = new IdentifiersExtractor($propertyNameCollectionFactory, $propertyMetadataFactory, $propertyAccessor);
$pathSegmentNameGenerator = new UnderscorePathSegmentNameGenerator(); 
$operationPathResolver = new OperationPathResolver($pathSegmentNameGenerator);
$subresourceOperationFactory = new SubresourceOperationFactory($resourceMetadataFactory, $propertyNameCollectionFactory, $propertyMetadataFactory, $pathSegmentNameGenerator);

class ApiLoader {
    private $resourceNameCollectionFactory;
    private $resourceMetadataFactory;
    private $identifiersExtractor;
    private $operationPathResolver;

    public function __construct(ResourceNameCollectionFactoryInterface $resourceNameCollectionFactory, ResourceMetadataFactoryInterface $resourceMetadataFactory, IdentifiersExtractorInterface $identifiersExtractor, OperationPathResolverInterface $operationPathResolver)
    {
        $this->resourceNameCollectionFactory = $resourceNameCollectionFactory;
        $this->resourceMetadataFactory = $resourceMetadataFactory;
        $this->identifiersExtractor = $identifiersExtractor;
        $this->operationPathResolver = $operationPathResolver;
    }

    public function load(): RouteCollection
    {
        $routeCollection = new RouteCollection();

        foreach ($this->resourceNameCollectionFactory->create() as $resourceClass) {
            $resourceMetadata = $this->resourceMetadataFactory->create($resourceClass);
            $resourceShortName = $resourceMetadata->getShortName();

            if (null === $resourceShortName) {
                throw new InvalidResourceException(sprintf('Resource %s has no short name defined.', $resourceClass));
            }

            if (null !== $collectionOperations = $resourceMetadata->getCollectionOperations()) {
                foreach ($collectionOperations as $operationName => $operation) {
                    $this->addRoute($routeCollection, $resourceClass, $operationName, $operation, $resourceMetadata, OperationType::COLLECTION);
                }
            }

            if (null !== $itemOperations = $resourceMetadata->getItemOperations()) {
                foreach ($itemOperations as $operationName => $operation) {
                    $this->addRoute($routeCollection, $resourceClass, $operationName, $operation, $resourceMetadata, OperationType::ITEM);
                }
            }
        }

        return $routeCollection;
    }

    private function addRoute(RouteCollection $routeCollection, string $resourceClass, string $operationName, array $operation, ResourceMetadata $resourceMetadata, string $operationType): void
    {
        $resourceShortName = $resourceMetadata->getShortName();

        if (isset($operation['route_name'])) {
            if (!isset($operation['method'])) {
                @trigger_error(sprintf('Not setting the "method" attribute is deprecated and will not be supported anymore in API Platform 3.0, set it for the %s operation "%s" of the class "%s".', OperationType::COLLECTION === $operationType ? 'collection' : 'item', $operationName, $resourceClass), E_USER_DEPRECATED);
            }

            return;
        }

        if (!isset($operation['method'])) {
            throw new RuntimeException(sprintf('Either a "route_name" or a "method" operation attribute must exist for the operation "%s" of the resource "%s".', $operationName, $resourceClass));
        }

        if (null === $controller = $operation['controller'] ?? null) {
            $controller = PlaceholderAction::class;
        }

        $operation['identified_by'] = (array) ($operation['identified_by'] ?? $resourceMetadata->getAttribute('identified_by', $this->identifiersExtractor ? $this->identifiersExtractor->getIdentifiersFromResourceClass($resourceClass) : ['id']));
        $operation['has_composite_identifier'] = \count($operation['identified_by']) > 1 ? $resourceMetadata->getAttribute('composite_identifier', true) : false;
        $path = trim(trim($resourceMetadata->getAttribute('route_prefix', '')), '/');
        $path .= $this->operationPathResolver->resolveOperationPath($resourceShortName, $operation, $operationType, $operationName);

        $route = new Route(
            $path,
            [
                '_controller' => $controller,
                '_format' => null,
                '_stateless' => $operation['stateless'],
                '_api_resource_class' => $resourceClass,
                '_api_identified_by' => $operation['identified_by'],
                '_api_has_composite_identifier' => $operation['has_composite_identifier'],
                sprintf('_api_%s_operation_name', $operationType) => $operationName,
            ] + ($operation['defaults'] ?? []),
            $operation['requirements'] ?? [],
            $operation['options'] ?? [],
            $operation['host'] ?? '',
            $operation['schemes'] ?? [],
            [$operation['method']],
            $operation['condition'] ?? ''
        );

        $routeCollection->add(RouteNameGenerator::generate($operationName, $resourceShortName, $operationType), $route);
    }
}

$apiLoader = new ApiLoader($resourceNameCollectionFactory, $resourceMetadataFactory, $identifiersExtractor, $operationPathResolver);
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

    public function match(string $pathinfo) {
        return $this->matcher->match($pathinfo);
    }

    public function setContext(RequestContext $context) {
        $this->context = $context;
    }

    public function getContext() {
        return $this->context;
    }

    public function generate(string $name, array $parameters = [], int $referenceType = self::ABSOLUTE_PATH) {
        return $this->generator->generate($name, $parameters, $referenceType);
    }
}

class ApiUrlGenerator implements ApiUrlGeneratorInterface {
    private $generator;

    public function __construct(UrlGeneratorInterface $generator)
    {
        $this->generator = $generator;
    }

    public function generate($name, $parameters = [], $referenceType = self::ABS_PATH) {
        return $this->generator->generate($name, $parameters, $referenceType ?: self::ABS_PATH);
    }
}

$apiUrlGenerator = new ApiUrlGenerator($generator);

$router = new Router($routes, $matcher, $generator, $requestContext);
$routeNameResolver = new RouteNameResolver($router);

$identifierDenormalizers = [new IntegerDenormalizer()];
$identifierConverter = new IdentifierConverter($identifiersExtractor, $propertyMetadataFactory, $identifierDenormalizers, $resourceMetadataFactory);

$iriConverter = new IriConverter($propertyNameCollectionFactory, $propertyMetadataFactory, $dataProvider, $routeNameResolver, $router, $propertyAccessor, $identifiersExtractor, /** SubresourceDataProviderInterface */ null, $identifierConverter, $resourceClassResolver, $resourceMetadataFactory);

$writeListener = new WriteListener($dataPersister, $iriConverter, $resourceMetadataFactory, $resourceClassResolver);

$serializerContextBuilder = new SerializerContextBuilder($resourceMetadataFactory);

$objectNormalizer = new ObjectNormalizer();

$nameConverter = new MetadataAwareNameConverter($classMetadataFactory);
$jsonLdContextBuilder = new JsonLdContextBuilder($resourceNameCollectionFactory, $resourceMetadataFactory, $propertyNameCollectionFactory, $propertyMetadataFactory, $apiUrlGenerator, $nameConverter);
$jsonLdItemNormalizer = new JsonLdItemNormalizer($resourceMetadataFactory, $propertyNameCollectionFactory, $propertyMetadataFactory, $iriConverter, $resourceClassResolver, $jsonLdContextBuilder, $propertyAccessor, $nameConverter, $classMetadataFactory, $defaultContext,  $dataTransformers, /** resource access checker **/ null);
$jsonLdObjectNormalizer = new JsonLdObjectNormalizer($objectNormalizer, $iriConverter, $jsonLdContextBuilder);
$jsonLdEncoder = new JsonLdEncoder('jsonld', new JsonEncoder());

$problemConstraintViolationListNormalizer = new ProblemConstraintViolationListNormalizer([], $nameConverter, $defaultContext);

$hydraCollectionNormalizer = new HydraCollectionNormalizer($jsonLdContextBuilder, $resourceClassResolver, $iriConverter, $defaultContext);
$hydraPartialCollectionNormalizer = new PartialCollectionViewNormalizer($hydraCollectionNormalizer, $configuration['collection']['pagination']['page_parameter_name'], $configuration['collection']['pagination']['enabled_parameter_name'], $resourceMetadataFactory, $propertyAccessor);
$hydraCollectionFiltersNormalizer = new CollectionFiltersNormalizer($hydraPartialCollectionNormalizer, $resourceMetadataFactory, $resourceClassResolver, $filterLocator);
$hydraErrorNormalizer = new HydraErrorNormalizer($apiUrlGenerator, $debug, $defaultContext);
$hydraEntrypointNormalizer = new HydraEntrypointNormalizer($resourceMetadataFactory, $iriConverter, $apiUrlGenerator);
$hydraDocumentationNormalizer = new HydraDocumentationNormalizer($resourceMetadataFactory, $propertyNameCollectionFactory, $propertyMetadataFactory, $resourceClassResolver, null, $apiUrlGenerator, /* SubresourceOperationFactoryInterface */ null, $nameConverter);
$hydraConstraintViolationNormalizer = new HydraConstraintViolationListNormalizer($apiUrlGenerator, [], $nameConverter);

$problemErrorNormalizer = new ErrorNormalizer($debug, $defaultContext);

$itemNormalizer = new ItemNormalizer($propertyNameCollectionFactory, $propertyMetadataFactory, $iriConverter, $resourceClassResolver, $propertyAccessor, $nameConverter, $classMetadataFactory, $dataProvider, $allowPlainIdentifiers, $logger, $dataTransformers, $resourceMetadataFactory, /** resourceAccessChecker **/ null);

$arrayDenormalizer = new ArrayDenormalizer();
$problemNormalizer = new ProblemNormalizer($debug, $defaultContext);
$jsonserializableNormalizer = new JsonSerializableNormalizer($classMetadataFactory, $nameConverter, $defaultContext);
$dateTimeNormalizer = new DateTimeNormalizer($defaultContext);
$dataUriNormalizer = new DataUriNormalizer();
$dateIntervalNormalizer = new DateIntervalNormalizer($defaultContext);
$dateTimeZoneNormalizer = new DateTimeZoneNormalizer();
$constraintViolationListNormalizer = new ConstraintViolationListNormalizer($defaultContext, $nameConverter);
$unwrappingDenormalizer = new UnwrappingDenormalizer($propertyAccessor);

$halItemNormalizer = new HalItemNormalizer($propertyNameCollectionFactory, $propertyMetadataFactory, $iriConverter, $resourceClassResolver, $propertyAccessor, $nameConverter, $classMetadataFactory, $dataProvider, $allowPlainIdentifiers, $defaultContext, $dataTransformers, $resourceMetadataFactory, /** resourceAccessChecker **/ null);

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
 * api_platform.jsonapi.normalizer.error                       -790       ApiPlatform\Core\JsonApi\Serializer\ErrorNormalizer
 * api_platform.jsonapi.normalizer.constraint_violation_list   -780       ApiPlatform\Core\JsonApi\Serializer\ConstraintViolationListNormalizer
 * api_platform.openapi.normalizer.api_gateway                 -780       ApiPlatform\Core\Swagger\Serializer\ApiGatewayNormalizer
 * api_platform.jsonapi.normalizer.entrypoint                  -800       ApiPlatform\Core\JsonApi\Serializer\EntrypointNormalizer
 * api_platform.jsonapi.normalizer.collection                  -985       ApiPlatform\Core\JsonApi\Serializer\CollectionNormalizer
 * api_platform.jsonapi.normalizer.item                        -890       ApiPlatform\Core\JsonApi\Serializer\ItemNormalizer
 * api_platform.jsonapi.normalizer.object                      -995       ApiPlatform\Core\JsonApi\Serializer\ObjectNormalizer
 */

$encoders = [new JsonEncoder(), $jsonLdEncoder];
$serializer = new Serializer(iterator_to_array($list), $encoders);

$serializeListener = new SerializeListener($serializer, $serializerContextBuilder, $resourceMetadataFactory);
$respondListener = new RespondListener($resourceMetadataFactory);
$formatListener = new AddFormatListener(new Negotiator(), $resourceMetadataFactory, $formats);
$readListener = new ReadListener($dataProvider, $dataProvider, /** SubresourceDataProvider **/ null, $serializerContextBuilder, $identifierConverter, $resourceMetadataFactory);
$deserializeListener = new DeserializeListener($serializer, $serializerContextBuilder, $formats, $resourceMetadataFactory);
$addLinkHeaderListener = new AddLinkHeaderListener($apiUrlGenerator);
$validationExceptionListener = new ValidationExceptionListener($serializer, $errorFormats, $exceptionToStatus);

$controller = new ExceptionAction($serializer, $errorFormats, $exceptionToStatus);
$errorListener = new ErrorListener($controller);
$exceptionListener = new ExceptionListener($controller, null, $debug, $errorListener);

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new RouterListener($matcher, new RequestStack()));
$dispatcher->addListener('kernel.view', [$validateListener, 'onKernelView'], 64);
$dispatcher->addListener('kernel.view', [$writeListener, 'onKernelView'], 32);
$dispatcher->addListener('kernel.view', [$serializeListener, 'onKernelView'], 16);
// TODO: ApiPlatform\Core\EventListener\QueryParameterValidateListener, prio 16   
$dispatcher->addListener('kernel.view', [$respondListener, 'onKernelView'], 8);
$dispatcher->addListener('kernel.request', [$formatListener, 'onKernelRequest'], 7);
$dispatcher->addListener('kernel.request', [$readListener, 'onKernelRequest'], 4);
$dispatcher->addListener('kernel.request', [$deserializeListener, 'onKernelRequest'], 2);
$dispatcher->addListener('kernel.exception', [$validationExceptionListener, 'onKernelException'], 2);
// $dispatcher->addListener('kernel.exception', [$exceptionListener, 'onKernelException'], -96);
$dispatcher->addListener('kernel.response', [$addLinkHeaderListener, 'onKernelResponse'], 2);

/*
 * TODO: 
 * api_platform.security.listener.request.deny_access     kernel.request      onSecurity                  3          ApiPlatform\Core\Security\EventListener\DenyAccessListener
 *   "                                                    kernel.request      onSecurityPostDenormalize   1                                                                   
 * api_platform.swagger.listener.ui                       kernel.request      onKernelRequest                        ApiPlatform\Core\Bridge\Symfony\Bundle\EventListener\SwaggerUiListener
 * api_platform.http_cache.listener.response.configure    kernel.response     onKernelResponse            -1         ApiPlatform\Core\HttpCache\EventListener\AddHeadersListener
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

$openApiFactory = new OpenApiFactory($resourceNameCollectionFactory, $resourceMetadataFactory, $propertyNameCollectionFactory, $propertyMetadataFactory, $jsonSchemaFactory, $jsonSchemaTypeFactory, $operationPathResolver, $filterLocator, $subresourceOperationFactory, $identifiersExtractor, $formats, $openApiOptions, $paginationOptions);
$documentationAction = new DocumentationAction($openApiFactory);
$routes->add('api_doc', new Route('/docs.{_format}', ['_controller' => $documentationAction, '_format' => null, '_api_respond' => true]));

$entryPointAction = new EntrypointAction($resourceNameCollectionFactory);
$routes->add('api_entrypoint', new Route('/{index}.{_format}', ['_controller' => $entryPointAction, '_format' => null, '_api_respond' => true, 'index' => 'index'], ['index' => 'index']));

$contextAction = new ContextAction($jsonLdContextBuilder, $resourceNameCollectionFactory, $resourceMetadataFactory);
$routes->add('api_jsonld_context', new Route('/contexts/{shortName}.{_format}', ['_controller' => $contextAction, '_format' => 'jsonld', '_api_respond' => true], ['shortName' => '.+']));

$controllerResolver = new ControllerResolver();
$argumentResolver = new ArgumentResolver();

$kernel = new HttpKernel($dispatcher, $controllerResolver, new RequestStack(), $argumentResolver);

$request = Request::createFromGlobals();
$response = $kernel->handle($request);
$response->send();
$kernel->terminate($request, $response);
```
