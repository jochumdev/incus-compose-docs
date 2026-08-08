---
date: 2026-08-08T01:56:03.000Z
dateCreated: 2026-07-12T02:06:29.777Z
tags: []
leafwiki_id: 1rVTw5YDg
leafwiki_title: DNS
leafwiki_created_at: "2026-07-12T02:06:29.777616934Z"
leafwiki_updated_at: "2026-08-08T01:56:03.000000000Z"
leafwiki_creator_id: vOmfrlBDg
leafwiki_last_author_id: vOmfrlBDg
---

# DNS

A split-horizon home resolver: [dnscrypt-proxy](https://dnscrypt.info/) as the client-facing resolver, [knot](https://www.knot-dns.cz/) as the authoritative server for the local `example.com` zone.

The files for this example are on [Github](https://github.com/lxc/incus-compose/tree/main/examples/dns).

## The example

| Service          | Role                                                                                                                                                 | Static IP      |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- |
| `dnscrypt-proxy` | Resolver clients point at. Forwards `example.com` (and `10.in-addr.arpa`) to `knot`; everything else goes out over DNSCrypt/DoH to public resolvers. | `10.179.215.2` |
| `knot`           | Authoritative DNS server for the `example.com` zone.                                                                                                 | `10.179.215.3` |

```mermaid
flowchart LR
    C[a client on your network] --> DP["dnscrypt-proxy<br/>10.179.215.2"]
    DP -->|"example.com<br/>10.in-addr.arpa"| K["knot<br/>10.179.215.3<br/>authoritative"]
    DP -->|everything else| PUB["public resolvers<br/>over DNSCrypt / DoH"]
```

Both attach to the pre-existing external `incusbr0` network, alongside [`caddy`](https://docs.incus-compose.org/examples/caddy), which serves the `*.example.com` sites this zone resolves to.

## Usage

```bash
incus-compose up
```

Point a client's DNS at `10.179.215.2`.

## Notes

- `knot`'s storage volume is bind-mounted with `x-incus: security.shifted: true` and `initial.uid`/`initial.gid: 53`, so the container's `knot` user (uid/gid 53) owns the files correctly under Incus's idmapped mounts.
