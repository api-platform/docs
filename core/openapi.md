# OpenAPI Specification Support (formerly Swagger)

API Platform natively supports the [OpenAPI](https://www.openapis.org/) API specification format.

![Screenshot](../symfony/images/swagger-ui-1.png)

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/open-api-spec?cid=apip"><img src="../symfony/images/symfonycasts-player.png" alt="OpenAPI screencast"><br>Watch the OpenAPI screencast</a></p>

The specification of the API is available at the `/docs.jsonopenapi` path.
By default, OpenAPI v3 is used.
You can also get an OpenAPI v3-compliant version thanks to the `spec_version` query parameter: `/docs.jsonopenapi?spec_version=3`

It also integrates a customized version of [Swagger UI](https://swagger.io/swagger-ui/) and [ReDoc](https://rebilly.github.io/ReDoc/), some nice tools to display the
API documentation in a user friendly way.

## Using the OpenAPI Command

You can also dump an OpenAPI specification for your API.

OpenAPI, JSON format:

```console
docker compose exec php \
    bin/console api:openapi:export
```

OpenAPI, YAML format:

```console
docker compose exec php \
    bin/console api:openapi:export --yaml
```

Create a file containing the specification:

```console
docker compose exec php \
    bin/console api:openapi:export --output=swagger_docs.json
```

If you want to use the old OpenAPI v2 (Swagger) JSON format, use:

```console
docker compose exec php \
    bin/console api:swagger:export
```

## Overriding the OpenAPI Specification

Symfony allows to [decorate services](https://symfony.com/doc/current/service_container/service_decoration.html), here we
need to decorate `api_platform.openapi.factory`.

In the following example, we will see how to override the title and the base path URL of the Swagger documentation and add a custom filter for
the `GET` operation of `/foos` path.

```yaml
# api/config/services.yaml
    App\OpenApi\OpenApiFactory:
        decorates: 'api_platform.openapi.factory'
        arguments: [ '@App\OpenApi\OpenApiFactory.inner' ]
        autoconfigure: false
```

```php
<?php
namespace App\OpenApi;

use ApiPlatform\OpenApi\Factory\OpenApiFactoryInterface;
use ApiPlatform\OpenApi\OpenApi;
use ApiPlatform\OpenApi\Model;

class OpenApiFactory implements OpenApiFactoryInterface
{

    public function __construct(private OpenApiFactoryInterface $decorated)
    {
    }

    public function __invoke(array $context = []): OpenApi
    {
        $openApi = $this->decorated->__invoke($context);
        $pathItem = $openApi->getPaths()->getPath('/api/grumpy_pizzas/{id}');
        $operation = $pathItem->getGet();

        $openApi->getPaths()->addPath('/api/grumpy_pizzas/{id}', $pathItem->withGet(
            $operation->withParameters(array_merge(
                $operation->getParameters(),
                [new Model\Parameter('fields', 'query', 'Fields to remove of the output')]
            ))
        ));

        $openApi = $openApi->withInfo((new Model\Info('New Title', 'v2', 'Description of my custom API'))->withExtensionProperty('info-key', 'Info value'));
        $openApi = $openApi->withExtensionProperty('key', 'Custom x-key value');
        $openApi = $openApi->withExtensionProperty('x-value', 'Custom x-value value');

        // to define base path URL
        $openApi = $openApi->withServers([new Model\Server('https://foo.bar')]);

        return $openApi;
    }
}
```

The impact on the swagger-ui is the following:

![Swagger UI](images/swagger-ui-modified.png)

## Using the OpenAPI and Swagger Contexts

Sometimes you may want to change the information included in your OpenAPI documentation.

For the full list of available configurations, please refer to the [OpenAPI Specifications](https://spec.openapis.org/oas/latest.html). The current doc page only gives some examples but focuses mostly on the OpenAPI integration inside API Platform without telling you all you can pass into the attributes.

The following configuration will give you total control over your OpenAPI definitions:

<code-selector>

```php
<?php
// api/src/Entity/Product.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\ApiProperty;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

#[ORM\Entity]
#[ApiResource]
class Product // The class name will be used to name exposed resources
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    /**
     * @param string $name A name property - this description will be available in the API documentation too.
     *
     */
    #[ORM\Column]
    #[Assert\NotBlank]
    #[ApiProperty(
        openapiContext: [
            'type' => 'string',
            'enum' => ['one', 'two'],
            'example' => 'one'
        ]
    )]
    public string $name;

    #[ORM\Column(type: "datetime")] 
    #[Assert\DateTime]
    #[ApiProperty(
        openapiContext: [
            'type' => 'string',
            'format' => 'date-time'
        ]
    )]
    public $timestamp;

    // ...
}
```

```yaml
# api/config/api_platform/properties.yaml
properties:
    App\Entity\Product:
        name:
            openapiContext:
                type: string
                enum: ['one', 'two']
                example: one
        timestamp:
            openapiContext:
                type: string
                format: date-time
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/properties.xml -->

<properties xmlns="https://api-platform.com/schema/metadata/properties-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/properties-3.0
           https://api-platform.com/schema/metadata/properties-3.0.xsd">
    <property resource="App\Entity\Product" name="name">
        <openapiContext>
            <values>
                <value name="type">type</value>
                <value name="enum">
                    <values>
                        <value>one</value>
                        <value>two</value>
                    </values>
                </value>
                <value name="example">one</value>
            </values>
        </openapiContext>
    </property>
    <property resource="App\Entity\Product" name="timestamp">
        <openapiContext>
            <values>
                <value name="type">string</value>
                <value name="format">date-time</value>
            </values>
        </openapiContext>
    </property>
</properties>
```

</code-selector>

This will produce the following Swagger documentation:

```json
"components": {
    "schemas": {
        "GrumpyPizza:jsonld": {
            "type": "object",
            "description": "",
            "properties": {
                "@context": {
                    "readOnly": true,
                    "type": "string"
                },
                "@id": {
                    "readOnly": true,
                    "type": "string"
                },
                "@type": {
                    "readOnly": true,
                    "type": "string"
                },
                "timestamp": {
                    "type": "string",
                    "format": "date-time"
                },
                "id": {
                    "readOnly": true,
                    "type": "integer"
                },
                "name": {
                    "type": "string",
                    "enum": [
                        "one",
                        "two"
                    ],
                    "example": "one"
                }
            }
        }
    }
}
```

To pass a context to the OpenAPI **v2** generator, use the `swaggerContext` attribute (notice the prefix: `swagger` instead of `openapi`).

## Disabling an Operation From OpenAPI Documentation

Sometimes you may want to disable an operation from the OpenAPI documentation, for example to not exposing it.
Using the `openapi` boolean option disables this operation from the OpenAPI documentation:

<code-selector>

```php
<?php
// api/src/Entity/Product.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;

#[ApiResource(
    operations: [
        new GetCollection(openapi: false)
    ]
)]
class Product
{
    // ...
}
```

```yaml
# api/config/api_platform/resources.yaml
resources:
    App\Entity\Product:
        operations:
            ApiPlatform\Metadata\GetCollection:
                openapi: false
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Product">
        <operations>
            <operation class="ApiPlatform\Metadata\GetCollection" openapi="false" />
        </operations>
    </resource>
</resources>
```

</code-selector>

Note: as your route is not exposed, you may want to return a HTTP 404 if it's called. Prefer using the `NotExposedAction` controller instead.

## Changing the Name of a Definition

API Platform generates a definition name based on the serializer `groups` defined
in the (`de`)`normalizationContext`. It's possible to override the name
thanks to the `openapi_definition_name` option:

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Post;

#[ApiResource]
#[Post(denormalizationContext: ['groups' => ['user:read'], 'openapi_definition_name' => 'Read'])]
class User
{
    // ...
}
```

It's also possible to re-use the (`de`)`normalizationContext`:

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Post;

#[ApiResource]
#[Post(denormalizationContext: [User::API_WRITE])]
class User
{
    const API_WRITE = [
        'groups' => ['user:read'],
        'openapi_definition_name' => 'Read',
    ];
}
```

## Changing Operations in the OpenAPI Documentation

You also have full control over both built-in and custom operations documentation.

<code-selector>

```php
<?php
// api/src/Entity/Rabbit.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Post;
use ApiPlatform\OpenApi\Model;
use App\Controller\RandomRabbit;

#[ApiResource]
#[Post(
    name: 'create_rabbit', 
    uriTemplate: '/rabbit/create', 
    controller: RandomRabbit::class, 
    openapi: new Model\Operation(
        summary: 'Create a rabbit picture', 
        description: '# Pop a great rabbit picture by color!\n\n![A great rabbit](https://rabbit.org/graphics/fun/netbunnies/jellybean1-brennan1.jpg)', 
        requestBody: new Model\RequestBody(
            content: new \ArrayObject([
                'application/json' => [
                    'schema' => [
                        'type' => 'object', 
                        'properties' => [
                            'name' => ['type' => 'string'], 
                            'description' => ['type' => 'string']
                        ]
                    ], 
                    'example' => [
                        'name' => 'Mr. Rabbit', 
                        'description' => 'Pink Rabbit'
                    ]
                ]
            ])
        )
    )
)]
class Rabbit
{
    // ...
}
```

```yaml
resources:
  App\Entity\Rabbit:
    operations:
      create_rabbit:
        class: ApiPlatform\Metadata\Post
        path: '/rabbit/create'
        controller: App\Controller\RandomRabbit
        openapi:
          summary: Random rabbit picture
          description: >
            # Pop a great rabbit picture by color!

            ![A great rabbit](https://rabbit.org/graphics/fun/netbunnies/jellybean1-brennan1.jpg)
          requestBody:
            content:
              application/json:
                schema:
                  type: object
                  properties:
                    name: { type: string }
                    description: { type: string }
                example:
                  name: Mr. Rabbit
                  description: Pink rabbit

```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
        https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Rabbit">
        <operations>
            <operation class="ApiPlatform\Metadata\Post" name="create_rabbit" uriTemplate="/rabbit/create"
                       controller="App\Controller\RandomRabbit">
                <openapi summary="Create a rabbit picture"
                         description="# Pop a great rabbit picture by color!!
    
    ![A great rabbit](https://rabbit.org/graphics/fun/netbunnies/jellybean1-brennan1.jpg)">
                    <responses>
                        <response status="200">
                            <content>
                                <values>
                                    <value name="application/json">
                                        <values>
                                            <value name="schema">
                                                <values>
                                                    <value name="type">object</value>
                                                    <value name="properties">
                                                        <values>
                                                            <value name="name">
                                                                <values>
                                                                    <value name="type">string</value>
                                                                </values>
                                                            </value>
                                                            <value name="description">
                                                                <values>
                                                                    <value name="type">string</value>
                                                                </values>
                                                            </value>
                                                        </values>
                                                    </value>
                                                </values>
                                            </value>
                                        </values>
                                    </value>
                                </values>
                            </content>
                        </response>
                    </responses>
                </openapi>
            </operation>
        </operations>
    </resource>
</resources>
```

</code-selector>

![Impact on Swagger UI](../symfony/images/swagger-ui-2.png)

## Disabling Swagger UI or ReDoc

To disable Swagger UI (ReDoc will be shown by default):

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    # ...
    enable_swagger_ui: false
```

To disable ReDoc:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    # ...
    enable_re_doc: false
```

## Changing the Location of Swagger UI

By default, the Swagger UI is available at the API location (when the HTML format is asked) and at the route `/docs`.

You may want to change its route and/or disable it at the API location.

### Changing the Route

Manually register the Swagger UI controller:

```yaml
# app/config/routes.yaml
api_doc:
    path: /api_documentation
    controller: api_platform.swagger_ui.action
```

Change `/api_documentation` to the URI you wish Swagger UI to be accessible on.

### Disabling Swagger UI at the API Location

To disable the Swagger UI at the API location, disable both Swagger UI and ReDoc:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    # ...
    enable_swagger_ui: false
    enable_re_doc: false
```

If you have manually registered the Swagger UI controller, the Swagger UI will still be accessible at the route you have chosen.

## Using a custom Asset Package in Swagger UI

Sometimes you may want to use a different [Asset Package](https://symfony.com/doc/current/reference/configuration/framework.html#packages) for the Swagger UI.
In this way you'll have more fine-grained control over the asset URL generations.
This is useful i.e. if you want to use different base path, base URL or asset versioning strategy.

Specify a custom asset package name:

```yaml
# config/packages/api_platform.yaml
api_platform:
    asset_package: 'api_platform'
```

Set or override asset properties per package:

```yaml
# config/packages/framework.yaml
framework:
    # ...
    assets:
        base_path: '/custom_base_path' # the default
        packages:
            api_platform:
                base_path: '/'
```

## Overriding the UI Template

As described [in the Symfony documentation](https://symfony.com/doc/current/templating/overriding.html), it's possible to override the Twig template that loads Swagger UI and renders the documentation:

```twig
{# templates/bundles/ApiPlatformBundle/SwaggerUi/index.html.twig #}
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>{% if title %}{{ title }} {% endif %}My custom template</title>
    {# ... #}
</html>
```

You may want to copy the [one shipped with API Platform](https://github.com/api-platform/core/blob/main/src/Symfony/Bundle/Resources/views/SwaggerUi/index.html.twig) and customize it.

## Compatibility Layer with Amazon API Gateway

[AWS API Gateway](https://aws.amazon.com/api-gateway/) supports OpenAPI partially, but it [requires some changes](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-known-issues.html).
API Platform provides a way to be compatible with Amazon API Gateway.

To enable API Gateway compatibility on your OpenAPI docs, add `api_gateway=true` as query parameter: `http://www.example.com/docs.jsonopenapi?api_gateway=true`.
The flag `--api-gateway` is also available through the command-line.

## OAuth

If you implemented OAuth on your API, you should configure OpenApi's authorization using API Platform's configuration:

```yaml
api_platform:
    oauth:
        # To enable or disable OAuth.
        enabled: false

        # The OAuth client ID.
        clientId: ''

        # The OAuth client secret.
        clientSecret: ''

        # The OAuth type.
        type: 'oauth2'

        # The OAuth flow grant type.
        flow: 'application'

        # The OAuth token url.
        tokenUrl: '/oauth/v2/token'

        # The OAuth authentication url.
        authorizationUrl: '/oauth/v2/auth'

        # The OAuth scopes.
        scopes: []
```

Note that `clientId` and `clientSecret` are being used by the SwaggerUI if enabled.

### Configure the OAuth Scopes Option

The `api_platform.oauth.scopes` option requires an array value with the scopes name and description. For example:

```yaml
api_platform:
    oauth:
        scopes:
            profile: "This scope value requests access to the End-User's default profile Claims, which are: name, family_name, given_name, middle_name, nickname, preferred_username, profile, picture, website, gender, birthdate, zoneinfo, locale, and updated_at."
            email: "This scope value requests access to the email and email_verified Claims."
            address: "This scope value requests access to the address Claim."
            phone: "This scope value requests access to the phone_number and phone_number_verified Claims."
```

**Note:** if you're using an OpenID Connect server (such as Keycloak or Auth0), the `openid` scope **must** be set according
to the [OpenID Connect specification](https://openid.net/specs/openid-connect-core-1_0.html).

## Info Object

The [info object](https://swagger.io/specification/#info-object) provides metadata about the API like licensing information or a contact. You can specify this information using API Platform's configuration:

```yaml
api_platform:

    # The title of the API.
    title: 'API title'

    # The description of the API.
    description: 'API description'

    # The version of the API.
    version: '0.0.0'

    openapi:
        # The contact information for the exposed API.
        contact:
            # The identifying name of the contact person/organization.
            name:
            # The URL pointing to the contact information. MUST be in the format of a URL.
            url:
            # The email address of the contact person/organization. MUST be in the format of an email address.
            email:
        # A URL to the Terms of Service for the API. MUST be in the format of a URL.
        termsOfService:
        # The license information for the exposed API.
        license:
            # The license name used for the API.
            name:
            # URL to the license used for the API. MUST be in the format of a URL.
            url:
```
