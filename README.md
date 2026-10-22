# maintenance-proxy

Caddy reverse proxy for a maintenance page. Traefik sends every request to this container with priority `1000000`. Caddy returns the content of one upstream page. The browser stays on the host it called. There is no redirect.

If the upstream answers with a redirect, Caddy fetches that target and returns the page. The response has no `Location` header.

The Caddyfile is baked into the image. The server does not mount a volume.

## Why it exists

On 21 October 2026, planning, documentation and the search for a way to take the customer sites offline started at 11:00 Europe/Berlin. This image is the switch that came out of that. Well before 16:00 Europe/Berlin a lot of sites were already down. Shortly before 16:00 (14:00 UTC) everything that could be taken down was down, more than 100 projects and sites, either through this Caddy container or by shutting the server off.

The reason was PHP2Shell, two bugs in the PHP runtime. Customer sites sit behind Traefik. Traefik talks to nginx, and nginx talks to PHP, so a request that reaches the app reaches the runtime.

| CVE | Bug | Impact |
| --- | --- | --- |
| [CVE-2026-109618](https://www.cve.org/CVERecord?id=CVE-2026-109618) | Out-of-memory corruption | Execute code without authentication as the PHP web worker. |
| [CVE-2026-109619](https://nvd.nist.gov/vuln/detail/cve-2026-109619) | Stale global state across requests | A use-after-free can disclose worker memory or execute code. |

Sources: [php2shell.com](https://www.php2shell.com/), [German Wikipedia](https://de.wikipedia.org/wiki/PHP2Shell).

I built this for the eRecht24 services. This is my private repository. The whole team could use the same image for every other customer.

The package exists so a normal deploy takes the sites down. The service goes into the project's compose file. A developer ships it with a staging deploy. That deploy is enough. There is no production deploy, and nobody has to log into the server and change Traefik by hand. Any developer can pull the switch, not only the two maintainers.

The router rule is `PathPrefix(/)` with priority `1000000` on the `https` entrypoint. There is no `Host()` check. Traefik picks the highest priority, so this one container wins over every application router on that Traefik. Production and staging share it, so both go down together. Caddy does not forward the request to nginx or PHP. It proxies one maintenance page and keeps the browser on the host it called. API clients that send `Accept: application/json` or `Content-Type: application/json` get `503` and `{"message":"maintenance"}`.

The login service below is the compose snippet. Remove the container and Traefik routes to the applications again.

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
