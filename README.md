# my-filebrowser

Self-hosted [FileBrowser Quantum](https://github.com/gtsteffaniak/filebrowser) behind Traefik, deployed to a VPS by GitHub Actions.

The app is served at `https://files.<DOMAIN>`.

## Layout

| Path | Purpose |
|---|---|
| `compose.yaml` | The `filebrowser` service, Traefik labels and volume mounts |
| `config.yaml.template` | FileBrowser config; `${DOMAIN}` is filled in at deploy time |
| `.env.example` | Required environment variables |
| `.github/workflows/main.yml` | Deploys over SSH on every push to `main` |
| `data/` | Runtime state: rendered `config.yaml` and the database (gitignored) |
| `files/` | The files FileBrowser serves (created on the server; gitignored) |

## Requirements

- Docker with Compose v2
- A running Traefik instance attached to the external Docker network `traefik-public`, with a `websecure` entrypoint and a `letsencrypt` certificate resolver
- A DNS record for `files.<DOMAIN>` pointing at the server

## Configuration

Copy `.env.example` to `.env` and fill it in:

| Variable | Required | Description |
|---|---|---|
| `DOMAIN` | yes | Base domain; the app is served at `files.<DOMAIN>` |
| `FILEBROWSER_ADMIN_PASSWORD` | yes | Password for the `admin` user (at least 8 characters) |
| `TZ` | no | Container timezone (default `Europe/Moscow`) |

`config.yaml.template` sets up password login only, disables sign-ups, serves a single source, `/files`, and trusts Traefik's `X-Forwarded-*` headers so FileBrowser sees real client IPs.

The config format is FileBrowser Quantum's, not the original FileBrowser's: `port`, `baseURL` and `externalUrl` go under `http:`, sources go under `server.sources`, and `signup` goes under `auth.methods.password`. Keep the top-level `server:` key. The current stable release (v2.0.1) crashes at startup without it, with `panic: interface conversion: interface {} is nil`.

## Running manually

```bash
cp .env.example .env   # then edit it
mkdir -p data files

# render the config the same way CI does
set -a; . ./.env; set +a
sed "s|\${DOMAIN}|${DOMAIN}|g" config.yaml.template > data/config.yaml

sudo chown -R 1000:1000 data files   # the container runs as uid 1000
docker compose up -d
```

For local debugging, uncomment the `ports` entry in `compose.yaml` to expose the app on `127.0.0.1:8080`.

## Deployment

Every push to `main` (or a manual run of the workflow) does the following on the server over SSH:

1. Clones the repo to `/opt/apps/<repo-name>` if it isn't there, then hard-resets it to `origin/main`
2. Writes `.env` from the `APP_ENV` secret
3. Renders `data/config.yaml` from the template and checks that `externalUrl` matches the domain
4. Sets ownership of `data/` and `files/` to uid 1000
5. Pulls the image, runs `docker compose up -d` and prunes old images

Because of the hard reset, don't edit tracked files directly on the server.

`docker compose up -d` only recreates the container when the image or `compose.yaml` changes. A deploy that only changes `config.yaml.template` leaves the container running the old config, so restart it on the server afterwards:

```bash
docker compose restart filebrowser
```

### GitHub secrets

| Secret | Value |
|---|---|
| `SSH_HOST` | Server hostname or IP |
| `SSH_USER` | SSH user (needs Docker access and permission to `chown`) |
| `SSH_PRIVATE_KEY` | Private key for that user |
| `APP_ENV` | Full contents of the production `.env` |

The server must also be able to run `git clone git@github.com:<owner>/<repo>.git`, so add a deploy key for the repository.

## Common commands

```bash
docker compose logs -f filebrowser   # follow logs
docker compose pull && docker compose up -d   # update the image
docker compose restart filebrowser   # reload data/config.yaml
docker compose down   # stop
```
