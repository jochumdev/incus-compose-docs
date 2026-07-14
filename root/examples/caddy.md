---
tags: []
leafwiki_id: fC-tQcYDg
leafwiki_title: Caddy
leafwiki_created_at: "2026-07-12T02:05:37.588008658Z"
leafwiki_updated_at: "2026-07-12T02:06:12.842507983Z"
leafwiki_creator_id: vOmfrlBDg
leafwiki_last_author_id: vOmfrlBDg
---

# Caddy

[Caddy](https://caddyserver.com) as a reverse-proxy front door — automatic HTTPS, one `Caddyfile`, no separate cert management.

The files for this example are on [Github](https://github.com/lxc/incus-compose/tree/main/examples/caddy).

## The example

A single `caddy` service on the pre-existing external `incusbr0` network (static IP `10.179.215.4`). Its `Caddyfile` (`caddy/Caddyfile`) serves a static site directly and reverse-proxies to the other host-facing examples in this repo by their static IPs:

| Domain                                                                                    | Backend                                                                                               |
| ----------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `example.com`, `www.example.com`                                                          | static site in `sites/example.com`                                                                    |
| `git.example.com`                                                                         | [`gitea`](https://docs.incus-compose.org/examples/gitea) at `10.136.32.17:3000`                       |
| `clock.example.com`                                                                       | [`kimai`](https://docs.incus-compose.org/examples/kimai) at `10.137.32.17:8001`                       |
| `immich.example.com`                                                                      | [`immich`](https://docs.incus-compose.org/examples/immich) at `10.131.32.17:2283`                     |
| `docker-registry.example.com`, `ghcr-registry.example.com`, `gitlab-registry.example.com` | [`oci-registry-cache`](https://docs.incus-compose.org/examples/oci-registry-cache)'s three registries |

The [`dns`](https://docs.incus-compose.org/examples/dns) example serves the authoritative `example.com` zone these domains resolve against.

## Usage

```bash
incus-compose up
```

Bring up whichever backend examples you want proxied, point DNS at them (see [`dns`](https://docs.incus-compose.org/examples/dns)), and browse to the domains above.

## Notes

- The service sleeps 5s before starting Caddy, to let the network interface come up first.
