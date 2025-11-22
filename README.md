# jedi-traefik

This project defines a Docker container that
starts [Traefik](https://traefik.io/) on port `80`
allowing it to act as a reverse proxy for several of my projects.

Once the container is running, the admin dashboard is available
at http://localhost:8080.

More docs to follow once I've got a better idea of how to use this.

To start the Docker container, run:

```shell
docker compose up -d
```