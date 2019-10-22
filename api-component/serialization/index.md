# The Serialization Process

## Overall Process

API Platform embraces and extends the Symfony Serializer Component to transform PHP entities in (hypermedia) API responses.

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/serializer?cid=apip"><img src="../../distribution/images/symfonycasts-player.png" alt="Serializer screencast"><br>Watch the Serializer screencast</a></p>

The main serialization process has two stages:

![Serializer workflow](images/SerializerWorkflow.png)

> As you can see in the picture above, an array is used as a man-in-the-middle. This way, Encoders will only deal with turning specific formats into arrays and vice versa. The same way, Normalizers will deal with turning specific objects into arrays and vice versa.
-- [The Symfony documentation](https://symfony.com/doc/current/components/serializer.html)

Unlike Symfony itself, API Platform leverages custom normalizers, its router and the [data provider](../fetching-and-persisting-data/data-providers.md) system to perform an advanced transformation. Metadata are added to the generated document including links, type information, pagination data or available filters.

The API Platform Serializer is extendable. You can register custom normalizers and encoders in order to support other formats. You can also decorate existing normalizers to customize their behaviors.

## Available Serializers

* [JSON-LD](https://json-ld.org) serializer
`api_platform.jsonld.normalizer.item`

JSON-LD, or JavaScript Object Notation for Linked Data, is a method of encoding Linked Data using JSON. It is a World Wide Web Consortium Recommendation.

* [HAL](https://en.wikipedia.org/wiki/Hypertext_Application_Language) serializer
`api_platform.hal.normalizer.item`

* JSON, XML, CSV, YAML serializer (using the Symfony serializer)
`api_platform.serializer.normalizer.item`

## The Serialization Context, Groups and Relations

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/serialization-groups?cid=apip"><img src="../../distribution/images/symfonycasts-player.png" alt="Serialization Groups screencast"><br>Watch the Serialization Groups screencast</a></p>

API Platform allows you to specify the `$context` variable used by the Symfony Serializer. 
This variable is an associative array that has a handy `groups` key allowing you to choose **which attributes of the resource are exposed during the normalization (read) and denormalization (write) processes**.

It relies on the [serialization (and deserialization) groups](https://symfony.com/doc/current/components/serializer.html#attributes-groups)
feature of the Symfony Serializer component.

In addition to groups, you can use any option supported by the Symfony Serializer. For example, you can use [`enable_max_depth`](https://symfony.com/doc/current/components/serializer.html#handling-serialization-depth)
to limit the serialization depth.
