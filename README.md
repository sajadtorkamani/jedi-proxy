# jedi-proxy

This project defines a Docker container that
starts [Traefik](https://traefik.io/) on port `80`
allowing it to act as a reverse proxy for several of my projects.

Once the container is running, the admin dashboard is available
at http://localhost:8080.

## Compose layout

| File                    | Role                                                                 |
| ----------------------- | -------------------------------------------------------------------- |
| `compose.yaml`          | What every environment shares: image, ports 80/443, Docker socket, network. |
| `compose.dev.yaml`      | Laptop: unauthenticated dashboard on 8080, mkcert certificates.       |
| `compose.prod.yaml`     | Server: Let's Encrypt (`le` resolver), dashboard behind basic auth.   |
| `compose.override.yaml` | A symlink to one of the two above. Gitignored, per machine.           |

Compose merges `compose.yaml` + `compose.override.yaml` automatically, so once
the symlink is in place a bare `docker compose …` targets the right environment
with no flags and no `COMPOSE_FILE`.

Without the symlink, `docker compose up -d` refuses with `no service selected`
rather than starting an unconfigured Traefik — every site on the machine goes
through this container, so the wrong config is an outage. See the comments in
`compose.yaml` for how that guard works, and why each environment file repeats
the full Traefik `command`.

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

### Point compose at the dev config

```shell
ln -s compose.dev.yaml compose.override.yaml
```

### Create `.env` file and set values

```shell
cp .env.example .env
```

The dashboard values in it are only read by the prod config, so the defaults
are fine locally.

### Create certificates

`certs/` is gitignored — the `.key.pem` files are private keys, so a fresh clone
starts empty and Traefik will log an error for every cert listed in
`dynamic/dev/certs.yml` that is missing. Regenerate the ones you need with
[`./bin/mkcert <host>`](#create-certificates-for-host).

### Visit dashboard

Open http://localhost:8080.

## Prod setup

```shell
git clone git@github.com:sajadtorkamani/jedi-proxy.git ~/jedi-proxy
cd ~/jedi-proxy
docker network create jedi-proxy
ln -s compose.prod.yaml compose.override.yaml
cp .env.example .env    # then set the dashboard hash and domain — see below
mkdir -p letsencrypt && touch letsencrypt/acme.json && chmod 600 letsencrypt/acme.json
```

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
docker compose up -d
```

Keep the value single-quoted: the hash contains `$`, which compose would
otherwise try to interpolate.

### Upgrading a checkout from the old `docker-compose.*.yml` layout

Before this layout, the environment was picked by `COMPOSE_FILE` in `.env`.
That variable takes precedence over `compose.override.yaml`, and the file it
names no longer exists, so remove it as part of the switch. On the server:

```shell
cd ~/jedi-proxy
git pull --ff-only
sed -i '/^COMPOSE_FILE=/d' .env
ln -s compose.prod.yaml compose.override.yaml
docker compose up -d --dry-run   # expect "Container jedi-proxy Running": nothing to recreate
docker compose up -d
```

Then check straight away — every site on the server goes through this container:

- a site's certificate is issued by Let's Encrypt, not `TRAEFIK DEFAULT CERT`
  (`echo | openssl s_client -connect <site>:443 -servername <site> 2>/dev/null | openssl x509 -noout -issuer`);
- the dashboard answers `401` without credentials;
- port 8080 is closed.

On the laptop it is the same, with `compose.dev.yaml` and `sed -i ''` (BSD sed).
Until the laptop is switched, a bare `docker compose` there (including other
projects' scripts that start the proxy) fails, because `COMPOSE_FILE` points at
a file that is gone.
