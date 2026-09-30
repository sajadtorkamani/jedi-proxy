# jedi-proxy

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

## Local setup (macOS)

### Create Docker network
```shell
docker network create jedi-proxy
```

### Install `mkcert` & `nss`

Visit the [docs](https://github.com/FiloSottile/mkcert) for installation
instructions for other platforms but on macOS, run:

```shell
brew install mkcert
brew install nss # if you use Firefox
```

### Create `.env` file and set values

```shell
cp .env.example .env
```

### Create certificates

`certs/` is gitignored — the `.key.pem` files are private keys, so a fresh clone
starts empty and Traefik will log an error for every cert listed in
`dynamic/dev/certs.yml` that is missing. Regenerate the ones you need with
[`./bin/mkcert <host>`](#create-certificates-for-host).

### Visit dashboard

Open http://localhost:8080.

## Prod setup
TODO

### Start Docker service

```shell
docker compose up -d
```

### Recipes

#### Create certificates for host

Suppose your app's hostname is `traefik-testapp-1.localhost`, you can create a
certificate for it via `mkcert` by running:

```shell
./bin/mkcert traefik-testapp-1.localhost
```

See [bin/mkcert](./bin/mkcert) for the underyling `mkcert` command.

Restart Docker service so the changes are picked up:

```shell
docker compose restart
```

### Create password for the admin dashboard on prod

`dynamic/prod/dashboard.yml` already supplies the `sajad:` username, so
`TRAEFIK_DASHBOARD_PWD_HASH` must be the bcrypt hash **alone**. `htpasswd`
prints `sajad:<hash>`; storing that whole line gives Traefik
`sajad:sajad:<hash>`, which it rejects — the dashboard router is dropped (404)
and the hash is written to the log every few seconds.

On the server, this prompts for the new password (so it stays out of shell
history) and writes just the hash into `.env`:

```shell
cd ~/jedi-proxy
hash="$(htpasswd -nBC 12 sajad | cut -d: -f2- | tr -d '\n')"
sed -i "s|^TRAEFIK_DASHBOARD_PWD_HASH=.*|TRAEFIK_DASHBOARD_PWD_HASH='$hash'|" .env
unset hash
docker compose -f docker-compose.prod.yml up -d
```

Name the prod file explicitly (or set `COMPOSE_FILE=docker-compose.prod.yml`
in the server's `.env`). A bare `docker compose up -d` follows `COMPOSE_FILE`,
and if that still says `docker-compose.dev.yml` it recreates the proxy with the
dev config — no Let's Encrypt resolver, the self-signed default certificate on
every site, and 8080 published. That took every site on prod2 down on
2026-09-30 until the proxy was recreated from the prod file.

Keep the value single-quoted: the hash contains `$`, which compose would
otherwise try to interpolate.

