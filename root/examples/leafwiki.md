---
tags: []
leafwiki_id: Mn_bwcYvg
leafwiki_title: LeafWiki
leafwiki_created_at: "2026-07-12T02:08:27.812762163Z"
leafwiki_updated_at: "2026-07-13T02:12:40.524720566Z"
leafwiki_creator_id: vOmfrlBDg
leafwiki_last_author_id: vOmfrlBDg
---

# LeafWiki

[LeafWiki](https://leafwiki.com/) — a self-hosted wiki as a single Go binary, Markdown + SQLite on disk, no external database.

The page you see is a LeafWiki with a Git backup to [jochumdev/incus-compose-docs](https://github.com/jochumdev/incus-compose-docs).

The files for this example are on [Github](https://github.com/lxc/incus-compose/tree/main/examples/leafwiki).

## The example

A single `wiki` service. Demonstrates:

- a bind-mounted SSH deploy key (`ssh.key`, a placeholder here) for LeafWiki's optional Git backup (`LEAFWIKI_GIT_BACKUP*` env vars)

All config comes from `.env`.

## Usage

```bash
incus-compose up
```

Open http://10.135.32.17:8080/
