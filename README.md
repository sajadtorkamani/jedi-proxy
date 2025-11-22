# jedi-traefik

This project defines a Docker container that
starts [Traefik](https://traefik.io/) on port `80`
allowing it to act as a reverse proxy for several of my projects.

Once the container is running, the admin dashboard is available
at http://localhost:8080.

More docs to follow once I've got a better idea of how to use this.

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
mkcert \
-key-file certs/traefik-testapp-1.key.pem \
-cert-file certs/traefik-testapp-1.cert.pem \
traefik-testapp-1.localhost
```


