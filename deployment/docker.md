# Using API Platform with Docker

API Platform projects can be run through [Docker](https://www.docker.com/).
A [Docker compose](https://docs.docker.com/compose/) configuration including a fully working [LAMP](https://en.wikipedia.org/wiki/LAMP_(software_bundle))
stack is shipped with the standard edition.

To install it, run the following commands (Docker must be installed on your system):

    docker-compose up # Download, build and run Docker images
    docker-compose run web composer install -o -n # Install Composer dependencies
    docker-compose run web bin/console doctrine:schema:create # Create the MySQL schema

Your project will be accessible at the URL `http://127.0.0.1`.

Previous chapter: [Deploying an API Platform App on Heroku](heroku.md)
