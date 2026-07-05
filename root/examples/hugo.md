---
tags: []
leafwiki_id: jiajRXfvR
leafwiki_title: Hugo
leafwiki_created_at: "2026-07-05T04:30:21.277561198Z"
leafwiki_updated_at: "2026-07-05T04:31:03.780610441Z"
leafwiki_creator_id: vOmfrlBDg
leafwiki_last_author_id: vOmfrlBDg
---
# Hugo

Example of a `build:` service: Hugo generates a static site, nginx-unprivileged serves it.

## Usage

```bash
incus-compose up
```

Open https://example.com - after you configured a proxy to handle it.:

## How it works

`compose.yaml` defines the build with `Dockerfile`:

1. `alpine` installs Hugo and builds `site/` into `public/`
2. `nginxinc/nginx-unprivileged:alpine` serves the output

## .env

```sh
BASE_URL="https://example.com"
```

## Dockerfile

```
ARG BASE_URL

FROM docker.io/alpine:latest AS builder

RUN apk add --no-cache hugo
WORKDIR /site
COPY site/ .
RUN hugo build -b "${BASE_URL}"

FROM docker.io/nginxinc/nginx-unprivileged:alpine
COPY --from=builder /site/public /usr/share/nginx/html
```

## compose.yaml

```yaml
services:
  blog:
    build:
      args:
        BASE_URL: "${BASE_URL}"
    environment:
      NGINX_ENTRYPOINT_WORKER_PROCESSES_AUTOTUNE: 1
    x-incus:
      limits.cpu: 1
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://127.0.0.1:8080"]
      interval: 5s
      timeout: 5s
      retries: 3
      start_period: 30s
      start_interval: 1s
```
