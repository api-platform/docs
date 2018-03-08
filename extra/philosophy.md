# API Platform's Philosophy

In 20 years of PHP, the web changed dramatically and is now evolving faster than ever:

* Now that [they are indexed by Google](http://searchengineland.com/tested-googlebot-crawls-javascript-heres-learned-220157)
  and thanks to awesome frontend technologies such as [AngularJS](http://angularjs.org/) or [React](https://facebook.github.io/react/),
  [full-Javascript](https://en.wikipedia.org/wiki/Single-page_application) **webapps are becoming the standard**.
* [Since 2014 internet users spend more time on their mobile devices than on desktops](http://techcrunch.com/2014/08/21/majority-of-digital-media-consumption-now-takes-place-in-mobile-apps/): having a responsive website is mandatory and **native mobile apps are a must-have**.
* [The semantic web](https://en.wikipedia.org/wiki/Semantic_Web) and **especially [Linked Data](http://en.wikipedia.org/wiki/Linked_data)
  is a reality**: with the [Schema.org](http://schema.org/) initiative and new open web standards such as [JSON-LD](http://json-ld.org/),
  search engines (among a bunch of other services and softwares) consume structured and machine-readable data at web scale.
  Not exposing such data decrease interoperability and search engine ranking/efficiency (think rich snippets).

[PHP.net](http://php.net), [Symfony](https://symfony.com), [Facebook](http://hhvm.com/) and many others have worked hard
to improve and professionalize the PHP ecosystem. The PHP world has closed the gap with most backend solutions and is often
more innovative than them.

However in critical area I've described previously, many things can be improved. Almost all existing solutions are still [designed
and documented](https://symfony.com/doc/current/book/page_creation.html) to create websites the old way: a server generates
then sends plain-old HTML documents to browsers.

[API Platform](https://api-platform.com) is a set of tools for building modern web projects. It is a framework
for API-first projects built on top of Symfony. Like other modern frameworks such as Zend Framework and Symfony, it's both
a full-stack all-in-one framework and a set of independent PHP components and bundles that can be used separately.

The architecture promoted by the framework will distrust many of you but read this tutorial until the end and you will
see how API Platform make modern development easy and fun again:

* [Start by **creating a hypermedia REST API**](../distribution/index.md) exposing structured data that can
  be understood by any compliant client such your apps but also as search engines (JSON-LD with Schema.org vocabulary).
  This API is the central and unique entry point to access and modify data. It also encapsulates the whole business logic.
* [Then **create as many clients as you want using frontend technologies you love**](../client-generator/index.md): an HTML5/Javascript
  webapp querying the API in AJAX (of course) but also a native iOS or Android app, or even a desktop application. Clients
  only display data and forms.
