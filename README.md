# jedi-traefik

This project defines a Docker container that
starts [Traefik](https://traefik.io/) on port `80`
allowing it to act as a reverse proxy for several of my projects.

Once the container is running, the admin dashboard is available
at http://localhost:8080.

More docs to follow once I've got a better idea of how to use this. For now,
I've added the below two projects as examples of how this single Traefik proxy
can be used to host multiple Dockerized applications:
- [traefik-testapp-1](https://github.com/sajadtorkamani/traefik-testapp-1): https://traefik-testapp-1.sajadtorkamani.com/
- [traefik-testapp-2](https://github.com/sajadtorkamani/traefik-testapp-2): https://traefik-testapp-2.sajadtorkamani.com/

## Getting started

### Install `mkcert` & `nss`

Visit the [docs](https://github.com/FiloSottile/mkcert) for installation
instructions for other platforms but on macOS, run:

```shell
brew install mkcert
brew install nss # if you use Firefox
```

To start the Docker container, run:

```shell
docker compose up -d
```

### Recipes

#### Create certificates for host

Suppose your app's hostname is `traefik-testapp-1.localhost`, you can run:

```shell
APP_HOST=traefik-testapp-1 && \
mkcert \
-key-file certs/$APP_HOST.key.pem \
-cert-file certs/$APP_HOST.cert.pem \
$APP_HOST.localhost
```

Update [dynamic/certs.yml](./dynamic/certs.yml) to reference the newly generated
certificate files.

