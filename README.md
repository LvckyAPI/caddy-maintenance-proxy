# maintenance-proxy

Caddy reverse proxy for a maintenance page. Traefik sends every request to this container with priority `1000000`. Caddy returns the content of one upstream page. The browser stays on the host it called. There is no redirect.

The Caddyfile is baked into the image. The server does not mount a volume.

## What it shows

`CADDY_UPSTREAM` is the origin, for example `https://www.e-recht24.de`.

`CADDY_UPSTREAM_PATH` is the path every request is rewritten to. The default is `/`.

Requests for files (`.css`, `.js`, `.svg`, and other short extensions) keep their path and are loaded from the same origin, so images and styles of that page still work.

## Local

Traefik must already be running. Set `TRAEFIK_NETWORK` to its Docker network. The local entrypoint is `http`.

```bash
cp .env.example .env
docker compose up -d --build
```

## Image

Pushing to `main` or a tag `v*` runs `.github/workflows/build-image.yml`. The workflow pushes the image to GitHub Container Registry:

`ghcr.io/<owner>/maintenance-proxy:latest`

Set `CADDY_IMAGE` to that address. On the server, log in once if the package is private:

```bash
echo "$GITHUB_TOKEN" | docker login ghcr.io -u USERNAME --password-stdin
```

## Server

The entrypoint is `https` with the `letsencrypt` certificate resolver. The container joins the external network `traefik-proxy`.

```bash
docker compose -f compose.production.yml pull
docker compose -f compose.production.yml up -d
```

Change the page by editing `.env` and recreating the container. The image stays the same.

```bash
docker compose -f compose.production.yml up -d
```

This router wins over every other router on that Traefik. Remove the container to put the other sites back.
