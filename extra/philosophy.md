# API Platform's Philosophy

In 25 years of PHP, the web changed dramatically and is now evolving faster than ever:

* Thanks to awesome frontend technologies such as [React](https://reactjs.org/) or [Vue.js](https://vuejs.org/),
  [full-JavaScript Progressive Web Apps](https://en.wikipedia.org/wiki/Progressive_web_application) **are becoming the standard**.
* [Internet users spend more time on their mobile devices than on desktops](https://www.broadbandsearch.net/blog/mobile-desktop-internet-usage-statistics): having a mobile-first website is mandatory and **native mobile apps are a must-have**.
* [The semantic web](https://en.wikipedia.org/wiki/Semantic_Web) and **especially [Linked Data](https://en.wikipedia.org/wiki/Linked_data)
  is a reality**: with the [Schema.org](http://schema.org/) initiative and new open web standards such as [JSON-LD](http://json-ld.org/),
  search engines (among a bunch of other services and software) consume structured and machine-readable data at web scale.
  Not exposing such data decrease interoperability and search engine ranking/efficiency (think rich snippets).
* HTTP/2 and HTTP/3 [dramatically improve the performance of web applications](https://vulcain.rocks) thanks to multiplexing, Server Push and their other new capabilities.

[PHP.net](https://www.php.net), [Symfony](https://symfony.com), [Facebook](http://hhvm.com/) and many others have worked hard
to improve and professionalize the PHP ecosystem. The PHP world has closed the gap with most backend solutions and is often
[more innovative](https://wiki.php.net/rfc) and [faster](https://benchmarksgame-team.pages.debian.net/benchmarksgame/fastest/php-python3.html) than them.

However in critical areas I've described previously, many things can be improved. Almost all existing solutions are still [designed
and documented](https://symfony.com/doc/current/book/page_creation.html) to create websites the old way: a server generates
then sends plain-old HTML documents to browsers.

[API Platform](https://api-platform.com) is a set of tools for building modern web projects. It is a framework
for API-first projects built on top of [Symfony components](https://symfony.com/projects/apiplatform).
Like other modern frameworks such as Laravel and Symfony, it's both a full-stack all-in-one framework and a set of independent PHP components and bundles that can be used separately.

API Platform makes modern development easy and fun again:

* [Start by **creating a web API**](../distribution/index.md) exposing structured data that can
  be understood by any compliant client such as your apps but also search engines (JSON-LD with Schema.org vocabulary).
  This API is the central and unique entry point to access and modify data. It also encapsulates the whole business logic.
* [Then **create as many clients as you want using frontend technologies you love**](../client-generator/index.md): a JavaScript
  webapp built in React or in Vue querying the API but also a native iOS or Android app, or even a desktop application. Clients
  only display data and forms.

See also [the general design](../core/design.md) of the framework.
