# Extending API Platform

API Platform's core functionalities are designed to provide the best experience and to offer compatibility with the different REST standards that are implemented, as well as GraphQL (see validation, pagination, authorization, filters, content negotiation, etc.).

On the contrary, custom behaviors implemented through Symfony controllers or through Symfony Kernel events must implement all these features by hand.

Therefore, it is discouraged to use custom Symfony controllers, or to plug subscribers or listeners onto Symfony Kernel events.

Instead, API Platform provides many ways in which you can customize the behavior of your API, described on this page.

## Custom Data Providers and Persisters

If you have specific data sources, the default Data Providers and Persisters for the Doctrine ORM, the Doctrine MongoDB ODM and Elasticsearch-PHP might not be enough for you. You might be relying on another database technology, or using a webservice to fetch and persist data.

In this case, you can create your own [Data Providers](./data-providers.md) and [Data Persisters](./data-persisters.md).

## Symfony Normalizer and Denormalizer

As explained in [The Serialization Process](./serialization.md), API Platform uses [The Symfony Serializer Component](https://symfony.com/doc/current/components/serializer.html) extensively.

If you need to customize the input or output objects on a global level in your API, you can customize the Serializer instance in the Symfony Container to hook custom logic to the process.

## Symfony Validation Constraints

If you need custom validation logic, you can have a look at [Symfony Validation Constraints](https://symfony.com/doc/current/reference/constraints.html).
