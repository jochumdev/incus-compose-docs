---
tags: []
leafwiki_id: rHHAQcLvR
leafwiki_title: Gitea
leafwiki_created_at: "2026-07-12T02:07:03.964655254Z"
leafwiki_updated_at: "2026-07-12T02:07:18.409911117Z"
leafwiki_creator_id: vOmfrlBDg
leafwiki_last_author_id: vOmfrlBDg
---

# Gitea

[Gitea](https://about.gitea.com/), a lightweight self-hosted Git service, backed by Postgres.

The files for this example are on [Github](https://github.com/lxc/incus-compose/tree/main/examples/gitea).

## The example

Two services: `database` (Postgres) and `gitea`, wired together with `depends_on: condition: service_healthy`. Versions, credentials, and network settings all come from `.env`.

## Usage

```bash
incus-compose up
```

Open http://10.136.32.17:3000/
