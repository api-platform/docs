# Using the Schema.org Vocabulary

API Platform Admin has native support for the popular [Schema.org](https://schema.org) vocabulary.

> Schema.org is a collaborative, community activity with a mission to create, maintain, and promote schemas for structured data on the Internet, on web pages, in email messages, and beyond.

To leverage this capability, your API must use the JSON-LD format and the appropriate Schema.org types.
The following examples will use [API Platform Core](../core/) to create such API, but keep in mind that this feature will work with any JSON-LD API using the Schema.org vocabulary, regardless of the used web framework or programming language.

## Displaying Related Resource's Name Instead of its IRI

By default, IRIs of related objects are displayed in lists and forms.
However, it is often more user-friendly to display a string representation of the resource (such as its name) instead of its ID.

To configure which property should be shown to represent your entity, map the property containing the name of the object with the `https://schema.org/name` type:

```php
// api/src/Entity/Person.php

#[ApiProperty(iri: "https://schema.org/name")]
private $name;
```

## Emails, URLs and Identifiers

Besides, it is also possible to use the documentation to customize some fields automatically while configuring the semantics of your data.

The following Schema.org types are currently supported by API Platform Admin:

* `https://schema.org/email`: the field will be rendered using the `<EmailField>` React Admin component
* `https://schema.org/url`: the field will be rendered using the `<UrlField>` React Admin component
* `https://schema.org/identifier`: the field will be formatted properly in inputs

Note: if you already use validation on your properties, the semantics are already configured correctly (see [the correspondence table](../core/validation.md#open-vocabulary-generated-from-validation-metadata))!
