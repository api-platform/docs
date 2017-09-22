# Using API Platform with Docker

API Platform projects can be run through [Docker](https://www.docker.com/).
A [Docker compose](https://docs.docker.com/compose/) configuration including a fully working [LAMP](https://en.wikipedia.org/wiki/LAMP_(software_bundle))
stack is shipped with the API Platform distribution.

## Services

The Docker Compose configuration comes with several ready-use services by default:

| Name    | Description                                                   | Port(s)
| ------- | ------------------------------------------------------------- | -------
| app     | The application with PHP and PHP-FPM 7.1, the latest Composer | N/A
| db      | A database provided by MySQL 5.7                              | N/A
| nginx   | An HTTP server provided by Nginx 1.11                         | 8080
| varnish | An HTTP cache provided by Varnish 4.1                         | 80

## Installation

To install it, run the following commands (Docker must be installed on your system):

    $ docker-compose up -d # Download, build and run Docker images
    $ docker-compose exec app bin/console doctrine:schema:create # Create the MySQL schema

Your project will be accessible in two different ways:
* Through the HTTP cache (Varnish): `http://localhost`
* Through the HTTP server directly (Nginx) to facilitate debugging: `http://localhost:8080`

Previous chapter: [Deploying an API Platform App on Heroku](heroku.md)

Next chapter: [API Platform Admin: Introduction](../admin/index.md)
