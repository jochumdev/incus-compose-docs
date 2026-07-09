---
date: 2026-07-05T01:34:08.995Z
dateCreated: 2026-07-05T01:20:36.522Z
description: Bring the familiar Docker Compose workflow to Incus containers. incus-compose implements the Compose specification for the Incus ecosystem, allowing you to define and run multi-container applications using the docker-compose.yml files you already know.
editor: markdown
published: true
tags: []
title: Home
leafwiki_id: iyelq_Bvg
leafwiki_title: Home
leafwiki_created_at: "2026-07-05T03:53:58.754411983Z"
leafwiki_updated_at: "2026-07-09T01:58:27.293884013Z"
leafwiki_creator_id: vOmfrlBDg
leafwiki_last_author_id: vOmfrlBDg
---
# incus-compose

## Why incus-compose?

[Incus](https://linuxcontainers.org/incus/) provides powerful system containers and virtual machines with superior security and isolation, but lacks the declarative multi-container orchestration that Docker Compose offers. This tool bridges that gap:

- Use existing `docker-compose.yml` files with Incus containers
- Leverage Incus's native OCI registry support for image pulling
- Run Docker/OCI images directly from registries
- Manage complex multi-container applications with familiar commands
- Benefit from Incus's resource efficiency and security model

## Demos

- [Beta 21 - many-dependencies 30 workers](https://asciinema.org/a/1260145) 
- [Beta 19 - Immich](https://asciinema.org/a/1259458)

## Features

Status: **Beta** - testing the beta release of incus-compose.

- Familiar commands: `up`, `down`, `start`, `stop`, `restart`, `list` (and `ps`), `logs`, `exec`, `config`, plus `build`, `healthd`, `incus` (pass-through), and `self-update`
- Compose project parsing via compose-go, with automatic `compose.incus.yaml` overrides and `x-incus` / `x-incus-compose` extensions for raw Incus options
- OCI image pulling from docker.io, ghcr.io, and other registries
- Two-stage image cache in a dedicated Incus project (survives `down`/`up`, avoids registry rate limits)
- Local image building via Podman/Docker [doc](/builds)
- Bridge networks with automatic name sanitization
- Static IPv4/IPv6 addresses with automatic DHCP ranges [doc](/compose-compatibility#automatic-dhcp-ranges)
- Port forwarding via proxy devices or kernel NAT mode [doc](/compose-compatibility#port-publishing)
- Storage volumes with UID/GID shifting; bind mounts (pass-through by default, optional seeding) [doc](/compose-compatibility#volume-permissions)
- Health checks, restart policies, and `depends_on: service_healthy` ordering via the `ic-healthd` sidecar [doc](/healthd)
- Service scaling with `up --scale` and orphan pruning
- Incus project isolation
- Resource limits and other advanced compose features (`shm_size`, `container_name`, etc.)
- Configuration via `INCUS_COMPOSE_*` environment variables for every flag, with a configurable parallel worker count [doc](/environment-variables)

## Quick Start

Requires `podman` or `docker` for image building and an Incus https remote (needed for healthchecking) with OCI registries added.
See [Getting Started](/getting-started) for the full setup walkthrough.

Install the latest release (the script verifies the SHA-256 checksum):

```bash
curl -sSfL https://raw.githubusercontent.com/lxc/incus-compose/main/install.sh | sh -s -- -b ~/.local/bin
```

Or grab a prebuilt archive from the [Releases Page](https://github.com/lxc/incus-compose/releases).

Then point it at your existing `compose.yaml`:

```bash
# Start services
incus-compose up

# View logs
incus-compose logs -f

# List running services
incus-compose list

# Stop and remove
incus-compose down
```

## Quick Links

- **[Contributing](https://github.com/lxc/incus-compose/blob/main/CONTRIBUTING.md)** - Contributing to incus-compose
- **[Changelog](https://github.com/lxc/incus-compose/blob/main/CHANGELOG.md)** - Changelog since 0.0.1-beta1

## Architecture

incus-compose uses a **resource-first design**, see [Architecture Documentation](/architecture) for details.

## Support and community

The following channels are available for you to interact with the Incus community.

### Bug reports

You can file bug reports and feature requests at: [`https://github.com/lxc/incus-compose/issues/new`](https://github.com/lxc/incus-compose/issues/new)

### Community support

Community support is handled at: [`https://discuss.linuxcontainers.org`](https://discuss.linuxcontainers.org)

## Contributing

Fixes and new features are greatly appreciated. Make sure to read our [contributing guidelines](https://github.com/lxc/incus-compose/blob/main/CONTRIBUTING.md) first!

## Credits

This project is inspired by [@bketelsen](https://github.com/bketelsen/incus-compose).
Some components are adapted from [docker compose](https://github.com/docker/compose).
The `install.sh` script is adapted from [golangci-lint](https://github.com/golangci/golangci-lint).

This project uses AI tools as development aids (drafting, iteration, reviews, tests, and documentation).
Architecture, constraints, and final code decisions are owned by the human committers.

Earlier development was on [Gitlab](https://gitlab.com/r3j0/incus-compose/).

## License

[Apache 2.0](https://github.com/lxc/incus-compose/blob/main/LICENSE)
