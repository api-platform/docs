# Migrate From FOSRestBundle

[FOSRestBundle](https://github.com/FriendsOfSymfony/FOSRestBundle) is a popular bundle to rapidly develop RESTful APIs with Symfony.
This page provides a guide to help developers migrating from FOSRestBundle to API Platform.

## Features Comparison

The table below provides a list of the main features you can find in FOSRestBundle 3.1, and their equivalents in API Platform.

### Make CRUD endpoints

**In FOSRestBundle**

Create a controller extending the `AbstractFOSRestController` abstract class, make your magic manually in your methods and return responses through the `handleView()` provided by FOSRest's `ControllerTrait`.

See [The view layer](https://github.com/FriendsOfSymfony/FOSRestBundle/blob/3.x/Resources/doc/2-the-view-layer.rst).

**In API Platform**

Add the `ApiResource` attribute to your entities, and enable operations you desire inside. By default, every operations are activated.

See [Operations](../operations.md).

### Make custom controllers

**In FOSRestBundle**

Same as above.

**In API Platform**

Even though this is not recommended, API Platform allows you to [create custom controllers](../controllers.md) and declare them in your entity's `ApiResource` attribute.

You can use them as you migrate from FOSRestBundle, but you should consider [switching to Symfony Messenger](../messenger.md) as it will give you more benefits, such as compatibility with both REST and GraphQL, and better performances of your API on big tasks.

See [General Design Considerations](../design.md).


### Routing system (with native documentation support)

**In FOSRestBundle**

Annotate your controllers with FOSRest's route annotations that are the most suitable to your needs.

See [Full default annotations](https://github.com/FriendsOfSymfony/FOSRestBundle/blob/3.x/Resources/doc/annotations-reference.rst).

**In API Platform**

Use the `ApiResource` attribute to activate the HTTP methods you need for your entity. By default, all the methods are enabled.

See [Operations](../operations.md).

### Hook into the requests handling

**In FOSRestBundle**

Listen to FOSRest's events to modify the requests before they come into your controllers, and the responses after they come out of them.

See [Listener support](https://github.com/FriendsOfSymfony/FOSRestBundle/blob/3.x/Resources/doc/3-listener-support.rst).

**In API Platform**

API Platform provides a lot of ways to customize the behavior of your API, depending on what you exactly want to do.

See [Extending API Platform](../extending.md) for more details.

### Customize the formats of the requests and the responses

**In FOSRestBundle**

Only the request body's format can be customized.

Use body listeners to use either FOSRest's own decoders or your own ones. FOSRestBundle provides native support for JSON and XML.

See [Body Listener](https://github.com/FriendsOfSymfony/FOSRestBundle/blob/3.x/Resources/doc/body_listener.rst).

**In API Platform**

Both the request and the response body's format can be customized.

You can configure the formats of the API either globally or in specific resources or operations. API Platform provides native support for multiple formats including JSON, XML, CSV, YAML, etc.

See [Content negociation](../content-negotiation.md).

### Name conversion

**In FOSRestBundle**

Only request bodies can be converted before entering into your controller.

FOSRest provides two native normalizers for converting the names of your JSON keys to camelCase. You can create your own ones by implementing the `ArrayNormalizerInterface`.

See [Body Listeners](https://github.com/FriendsOfSymfony/FOSRestBundle/blob/3.x/Resources/doc/body_listener.rst).

**In API Platform**

Both request and response bodies can be converted.

API Platform uses [name converters](https://symfony.com/doc/current/components/serializer.html#component-serializer-converting-property-names-when-serializing-and-deserializing) included in the Serializer component of Symfony. You can create your own by implementing the `NameConverterInterface` provided by Symfony.

See [_Name Conversion_ in The Serialization Process](../serialization.md#name-conversion).

### Handle errors

**In FOSRestBundle**

Map the exceptions to HTTP statuses in the `fos_rest.exception` parameter.

See [ExceptionController support](https://github.com/FriendsOfSymfony/FOSRestBundle/blob/3.x/Resources/doc/4-exception-controller-support.rst).

**In API Platform**

Map the exceptions to HTTP statuses in the `api_platform.exception_to_status` parameter.

See [Errors Handling](../errors.md).

### Security

**In FOSRestBundle**

Use [Symfony's Security component](https://symfony.com/doc/current/security) to control your API access.

**In API Platform**

Use the `security` attribute in the `ApiResource` and `ApiProperty` attributes. It is an [Expression language](https://symfony.com/doc/current/components/expression_language.md) string describing who can access your resources or who can see the properties of your resources. By default, everything is accessible without authentication.

Note you can also use the `security.yml` file if you only need to limit access to specific roles.

See [Security](../security.md).

### API versioning

**In FOSRestBundle**

FOSRestBundle provides a way to provide versions to your APIs in a way users have to specify which one they want to use.

See [API versioning](https://github.com/FriendsOfSymfony/FOSRestBundle/blob/3.x/Resources/doc/versioning.rst).

**In API Platform**

API Platform has no native support to API versioning, but instead provides an approach consisting of deprecating resources when needed. It allows a smoother upgrade for clients, as they need to change their code only when it is necessary.

See [Deprecating Resources and Properties](../deprecations.md).
