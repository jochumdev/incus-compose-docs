---
tags: []
leafwiki_id: 3NVYQcYvg
leafwiki_title: WikiJS
leafwiki_created_at: "2026-07-12T02:09:45.758010229Z"
leafwiki_updated_at: "2026-07-12T02:09:54.737814553Z"
leafwiki_creator_id: vOmfrlBDg
leafwiki_last_author_id: vOmfrlBDg
---

# wikijs

[Wiki.js](https://js.wiki/), a modern wiki app, backed by Postgres.

The files for this example are on [Github](https://github.com/lxc/incus-compose/tree/main/examples/wikijs).

## The example

Two services: `database` (Postgres) and `wiki`, wired together with `depends_on: condition: service_healthy`. Versions and credentials come from `.env`.

## Usage

```bash
incus-compose up
```

Open http://10.133.32.17:3000/

## Notes

- `init: true` on `wiki` isn't implemented by incus-compose yet — left in `compose.yaml` as documentation of intent.
