---
tags: []
leafwiki_id: 7BiJw5Yvg
leafwiki_title: Immich
leafwiki_created_at: "2026-07-12T02:07:30.949083077Z"
leafwiki_updated_at: "2026-07-12T02:46:17.884161633Z"
leafwiki_creator_id: vOmfrlBDg
leafwiki_last_author_id: vOmfrlBDg
---

# Immich

[Immich](https://immich.app/), a self-hosted photo and video backup solution.

The files for this example are on [Github](https://github.com/lxc/incus-compose/tree/main/examples/immich).

## The example

Five services, following [Immich's official Compose layout](https://docs.immich.app/install/docker-compose): `server`, `machine-learning`, `microservices` (background workers, split from `server` via `IMMICH_WORKERS_INCLUDE`/`EXCLUDE`), `redis`, and `database` (a Postgres fork with vector search support). Version, secrets, and storage paths come from `.env`.

## Usage

```bash
incus-compose up
```

Open http://10.131.32.17:2283/

## Notes

- `library`'s storage pool comes from `UPLOAD_POOL` in `.env` via `x-incus-compose.pool` — set it before first run if you need another pool than the default.
