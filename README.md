# maintenance-proxy

Caddy reverse proxy for a maintenance page. Traefik sends every request to this container with priority `1000000`. Caddy returns the content of one upstream page. The browser stays on the host it called. There is no redirect.

If the upstream answers with a redirect, Caddy fetches that target and returns the page. The response has no `Location` header.

The Caddyfile is baked into the image. The server does not mount a volume.

## What it shows

`CADDY_UPSTREAM` is the origin, for example `https://www.e-recht24.de`.

`CADDY_UPSTREAM_PATH` is the path every request is rewritten to. The default is `/`.

Requests for files (`.css`, `.js`, `.svg`, and other short extensions) keep their path and are loaded from the same origin, so images and styles of that page still work.

A request that asks for JSON gets JSON instead of that page. Caddy matches `Accept: application/json` or `Content-Type: application/json` and answers `503` with `{"message":"maintenance"}`. The browser request for the HTML page does not send that header, so it still receives the upstream page.

## Local

Traefik must already be running. Set `TRAEFIK_NETWORK` to its Docker network. The local entrypoint is `http`.

```bash
cp .env.example .env
docker compose up -d --build
```

## Image

The image for every environment is:

`ghcr.io/lvckyapi/caddy-maintenance-proxy:latest`

Pushing to `main` or a tag `v*` runs `.github/workflows/build-image.yml` and pushes that image to GitHub Container Registry. On the server, log in once if the package is private:

```bash
echo "$GITHUB_TOKEN" | docker login ghcr.io -u USERNAME --password-stdin
```

## Server

The entrypoint is `https` with the `letsencrypt` certificate resolver. The container joins the external network `traefik-proxy`.

The login service adds it like this in `compose.production.yml`. The priority `1000000` puts this router in front of the app:

```yaml
  caddy:
    image: ghcr.io/lvckyapi/caddy-maintenance-proxy:latest
    container_name: "${PROJECT_NAME}_caddy"
    restart: unless-stopped
    environment:
      CADDY_UPSTREAM: ${CADDY_UPSTREAM:-https://www.e-recht24.de}
      CADDY_UPSTREAM_PATH: ${CADDY_UPSTREAM_PATH:-/}
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.${PROJECT_NAME}_caddy.rule=PathPrefix(`/`)"
      - "traefik.http.routers.${PROJECT_NAME}_caddy.entrypoints=https"
      - "traefik.http.routers.${PROJECT_NAME}_caddy.tls=true"
      - "traefik.http.routers.${PROJECT_NAME}_caddy.tls.certresolver=letsencrypt"
      - "traefik.http.routers.${PROJECT_NAME}_caddy.priority=1000000"
      - "traefik.http.services.${PROJECT_NAME}_caddy.loadbalancer.server.port=80"
    networks:
      - traefik-proxy
```

```bash
docker compose -f compose.production.yml pull
docker compose -f compose.production.yml up -d
```

Change the page by editing `.env` and recreating the container. The image stays the same.

```bash
docker compose -f compose.production.yml up -d
```

This router wins over every other router on that Traefik. Remove the container to put the other sites back.
