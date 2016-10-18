# Troubleshooting

This is a list of common pitfalls on using API Platform, and how to avoid them.

## Using Docker

If the web container cannot start and display this `Error starting userland proxy: Bind for 0.0.0.0:80`, it means that the port 80 is already used.

The configuration docker is planned to be launch on the port 80 in the `docker-compose.yml` file.