# Send in Docker compose

This repository provides a basic Docker compose configuration to host a public
[Send](https://gitlab.com/timvisee/send) instance on your own domain.

- Hosts on your own domain
- Provides automatic SSL certificates through LetsEncrypt

This configuration exposes a reverse proxy on ports 80 and 443, so these must be
available.

This uses the latest Send version from
[`timvisee/send`](https://gitlab.com/timvisee/send) because
[`mozilla/send`](https://github.com/mozilla/send) has been archived.
This is configurable in your [`.env`](.env.example) file.

See [docker-compose.yaml](./docker-compose.yaml).

*Note: for plain Docker usage without Compose, see: https://github.com/timvisee/send/blob/master/docs/docker.md*

## Usage

1. Install Docker Engine with the Compose plugin: https://docs.docker.com/engine/install/
2. Clone this repository: `git clone https://github.com/timvisee/send-docker-compose && cd send-docker-compose`
3. Copy `.env.example` to `.env` and replace the host, email, image, and storage settings.
4. Create the host upload directory and ensure the Docker daemon can write it: `sudo install -d -m 750 /var/lib/send/uploads`
5. Validate the rendered configuration without starting containers. Because the
   Compose file loads an untracked `.env`, use a temporary copy of the
   sanitized sample: `cp .env.example .env && docker compose config --quiet && rm .env`
6. Start the stack in the background: `docker compose up -d`
7. Inspect startup and certificate logs with `docker compose logs -f send proxy-letsencrypt`, then visit `https://send.example.com`.

<img src="https://i.imgur.com/eyvrWAP.png" alt="Screenshot of succesfully running Send UI" width="450px"/>

The services use `restart: unless-stopped`, so they return after a Docker host
restart unless an operator intentionally stopped them. Stop the stack with
`docker compose down`; named Redis and certificate volumes are retained.

The Send container is intentionally not published on a host port. Only the
nginx proxy publishes ports 80 and 443. The upload bind mount and Redis volume
are persistent; install [gc.cron](./gc.cron) on the host for a periodic cleanup
fallback if an old upload is missed by the application.

## Example environment

The complete sanitized template is in [`.env.example`](./.env.example). Copy
it to `.env`, then replace the hostname, certificate email, image tag, and any
selected storage settings. The untracked `.env` is loaded into the Send
container by Compose; Compose-only values are kept in the file comments.

## Configuration

All the config options and their defaults can be found here: https://github.com/timvisee/send/blob/master/server/config.js

For more documentation about the config options available and their defaults, see: https://github.com/timvisee/send/blob/master/docs/docker.md

The sample supports local filesystem storage, AWS/S3-compatible storage, and
Google Cloud Storage. Set exactly one backend: leave both bucket variables
empty for the persistent host directory, or configure the selected object
store credentials through a secret manager or the untracked `.env` file.
`AWS_REGION` is required when `S3_BUCKET` is set. GCS uses Application Default
Credentials; do not commit a service-account key.

Before production use, the operator must choose a pinned Send image tag or
digest instead of `latest`, set the real `HOST` and `LETSENCRYPT_EMAIL`, and
choose exactly one storage model (the host filesystem, S3-compatible storage,
or GCS). Keep those values in the untracked `.env`; never add credentials to
the repository.

Config options expecting array values (e.g. `EXPIRE_TIMES_SECONDS`, `DOWNLOAD_COUNTS`) should be set as bare comma separated values, and the first entry is used as the default.

Other options should be set as unquoted strings, integers, booleans, etc. The
Compose service loads them from `.env`, so there is no need to edit the YAML
environment block. For example:
```dotenv
BASE_URL=https://send.example.com
MAX_DOWNLOADS=250000
MAX_EXPIRE_SECONDS=31536000
EXPIRE_TIMES_SECONDS=86400,3600,604800,2592000
DOWNLOAD_COUNTS=10,1,2,5,10,15,25,100
```
