# The API Platform Admin

<video controls autoplay playsinline muted loop>
  <source src="./images/admin-demo.mp4" type="video/mp4"/>
  <source src="./images/admin-demo.webm" type="video/webm"/>
  Sorry, your browser doesn't support HTML5 video.
</video>

API Platform **Admin** is a tool to automatically create a beautiful (Material Design) and fully-featured administration interface
for any API implementing specification formats supported by [`@api-platform/api-doc-parser`](https://github.com/api-platform/api-doc-parser).

In particular, that includes:

- APIs using [the Hydra Core Vocabulary](https://www.hydra-cg.com/)
- APIs exposing an [OpenAPI documentation](https://www.openapis.org/)

Of course, API Platform Admin is the perfect companion of APIs created
using [the API Platform framework](https://api-platform.com). But it also supports APIs written with any other programming language or framework as long as they expose a standard Hydra or OpenAPI documentation.

## Based On React Admin

API Platform Admin is a Single Page Application (SPA), based on [React Admin](https://marmelab.com/react-admin/), a powerful frontend framework for building B2B applications on top of REST/GraphQL APIs, written in TypeScript and React.

Thanks to its built-in **guessers**, API Platform Admin parses the API documentation then uses React Admin to expose a nice, responsive management interface (Create-Retrieve-Update-Delete, i.e. CRUD) for all documented resource types.

Afterwards, you can **customize everything** by using the numerous components provided [React Admin](https://marmelab.com/react-admin/documentation.html) and [MUI](https://mui.com/), or even writing your own [React](https://reactjs.org/) components.

<iframe src="https://www.youtube-nocookie.com/embed/UyAkN85wGNk" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen style="aspect-ratio: 16 / 9;width:100%;margin-bottom:1em;"></iframe>

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/react-admin?cid=apip"><img src="../symfony/images/symfonycasts-player.png" alt="React Admin Screencast"><br>Watch the React Admin screencast</a></p>

## Features

Simply by reading your API documentation, API Platform Admin provides the following features:

- Generate 'list', 'create', 'show', and 'edit' views for all resources
- Automatically detect the type for inputs and fields
- Client-side [validation](./validation.md) on required inputs
- Pagination
- Filtering and ordering
- Easily view and edit [related records](./handling-relations.md)
- Display the related resource’s name instead of its IRI ([using the Schema.org vocabulary](./schema-org.md#displaying-related-resources-name-instead-of-its-iri))
- Nicely displays server-side errors (e.g. advanced validation)
- Real-time updates with [Mercure](https://mercure.rocks)

By [leveraging React Admin components](./advanced-customization.md), you can further customize the generated interface and get access to many more features:

- Powerful Datagrid components
- Search and filtering
- Advanced form validation
- Undoable mutations
- Authentication
- Access Control
- Internationalization
- [And many more](https://marmelab.com/react-admin/Features.html)

## Next Step

Get your Admin up and running by following the [Getting Started guide](./getting-started.md).