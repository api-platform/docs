# Using API Platform with Docker

Projects using API Platform can be run through [Docker](https://www.docker.com/) and
[Docker compose](https://docs.docker.com/compose/). Run `docker-compose up -d`,
your project will be accessible at [http://127.0.0.1](http://127.0.0.1).

You can customize Docker configuration by creating your own `docker-compose.yml`
file. Form example if you want Nginx to run on port 8888:

```yaml
web:
    image: vincentchalamon/docker-symfony
    ports:
        - 8888:80
    volumes:
        - .:/var/www
    tty: true

db:
    image: mysql
    command: mysqld --port 3386
    net: "host"
    environment:
        MYSQL_DATABASE: api
        MYSQL_USER: root
        MYSQL_ALLOW_EMPTY_PASSWORD: yes
```
