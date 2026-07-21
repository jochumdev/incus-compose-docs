---
date: 2026-07-05T01:27:16.661Z
dateCreated: 2026-07-05T01:03:24.052Z
description: null
editor: markdown
published: true
tags: []
title: Why Incus?
leafwiki_id: GDzu3_fDR
leafwiki_title: Why Incus?
leafwiki_created_at: "2026-07-05T03:54:00.356119734Z"
leafwiki_updated_at: "2026-07-16T01:24:55.933702447Z"
leafwiki_creator_id: vOmfrlBDg
leafwiki_last_author_id: vOmfrlBDg
---
# Why Incus?

[Incus](https://linuxcontainers.org/incus/) is the best-kept secret in Linux
infrastructure: one daemon, one clean REST API, and it runs **OCI application
containers, LXC system containers, and full virtual machines** side by side.
It is developed in the open under the [Linux Containers](https://linuxcontainers.org/)
project by the team that has been building container technology for Linux since
LXC in 2008 - no vendor lock-in, no license surprises, just Apache-2.0 software
maintained by the people who know this space best.

There's one gap: most compose tooling assumes a Docker-style OCI engine.
`incus-compose` closes it. Your existing `compose.yaml`, running natively on
Incus.

## One Package, One Daemon

Think about what a typical container-plus-VM setup accumulates: an engine
daemon, a container runtime under it, a hypervisor manager beside it, network
plugins, storage drivers, and a different CLI for each. Incus replaces the
whole stack with **a single package and a single daemon** that already does
everything:

- **Containers and VMs** - application containers, system containers, and
  KVM virtual machines from the same command
- **Networking built in** - managed bridges with DHCP and DNS, no SDN add-on
- **Storage built in** - Ceph, ZFS, Btrfs, LVM, or plain directory pools, with
  snapshots and clones as first-class operations
- **Images, projects, and clustering** - image cache, multi-tenancy, and
  multi-host scaling, all in the same daemon

Install one package, and there is nothing else to run, patch, or babysit.
That lightness is a feature you feel every day.

## The Problem: Engines Inside Containers

Running an OCI engine inside an Incus container is a common workaround, but
you pay for it:

```mermaid
flowchart TB
    subgraph container["Incus Container"]
        subgraph engine["OCI Engine Daemon"]
            app["Your App"]
        end
    end
```

- Double overhead - two container runtimes doing one job
- Nested namespaces add complexity and failure modes
- Privileged nested containers weaken your security posture
- Layered filesystems inside layered filesystems waste storage

## The Solution: Run Images Natively

Incus can run OCI images directly - the app is PID 1, no init system and no
second engine in between. `incus-compose` drives exactly that mode:

```mermaid
flowchart TB
    subgraph incus["Incus"]
        app["Your App"]
    end
```

- One layer of containerization instead of two
- Native Incus security: unprivileged by default, AppArmor and seccomp confined
- The same compose files you already use
- And when an app genuinely needs a full OS environment, a system container
  with real init is one config line away

## What You Gain

| Feature        | OCI Engines                 | Incus                                             |
| -------------- | --------------------------- | ------------------------------------------------- |
| Container type | Application (PID 1 = app)   | Application (PID 1 = app) or system (full init)   |
| Isolation      | Namespaces + cgroups        | Namespaces + cgroups, unprivileged by default     |
| Security       | Varies by engine and config | AppArmor + seccomp confinement by default         |
| Networking     | Port mapping via iptables   | Real IPs and port proxies                         |
| Storage        | Overlay filesystem          | ZFS/Btrfs with instant snapshots (pool-dependent) |
| Image caching  | Per-engine cache            | Global blob cache, per-project alias              |

Real IPs deserve a special mention: every container gets its own network
address, so two services can both listen on port 80 without a port-mapping
puzzle. Shell into any container for debugging, snapshot it before a risky
upgrade, roll back in seconds - this is infrastructure that works *with* you.

And your compose file inherits these powers too: project-wide resource
limits, static IPs, GPU passthrough, storage-pool placement, and the full
Incus API via `x-incus`. See the feature overview on the
[home page](/home) and the complete matrix in
[Compose Compatibility](/compose-compatibility).

## Your Laptop Is Just a Thin Client

Incus is client/server. The daemon is Linux-only, but the `incus` client (and
`incus-compose`) is a cross-platform Go binary. From a Windows or macOS desktop
you connect to a remote Linux host over HTTPS and manage OCI app containers,
system containers, and full VMs - all without Docker Desktop, WSL, or a local
Linux VM.

Docker Desktop cannot do this: on Windows and macOS it runs a hidden Linux VM
to host the engine. With Incus the workload lives on real Linux infrastructure
and your laptop stays light.

See [Installing on Windows](/getting-started/windows) for the client setup.

## Grows With You: Laptop to Cluster

The same API that runs your dev stack scales to a data center - no rewrite, no
new orchestration layer to learn.

**[Incus clustering](https://linuxcontainers.org/incus/docs/main/explanation/clustering/):**

- Scale from 1 to 100+ bare metal hosts
- Single API endpoint for the entire cluster
- Automatic instance placement and load balancing
- Live migration between hosts

**[IncusOS](https://linuxcontainers.org/incus-os/):**

- Immutable OS purpose-built for Incus
- Safe, predictable updates
- Minimal attack surface
- Production-ready out of the box

## Is Incus Right for You?

**Choose Incus when:**

- You need to shell into containers for debugging
- You want true RW volumes (not Kubernetes volume limitations)
- You need real network addresses (no port conflicts)
- You want unprivileged-by-default containers without full VM overhead
- You need ZFS/Btrfs snapshots and clones
- You're running apps that expect a full OS environment
- Security and multi-tenancy are priorities
- You're already using Incus for infrastructure
- You want one workflow from dev laptop to production cluster

**Stick with OCI engines when:**

- You're targeting Kubernetes deployment
- You need the absolute broadest ecosystem compatibility - base images, CI
  templates, and marketplace integrations mostly assume Docker/OCI
- You want a managed cloud container service (ECS, Cloud Run, GKE Autopilot)
  instead of operating your own hosts
- You're relying on the depth of existing tutorials, Stack Overflow answers,
  and community troubleshooting that comes with Docker's larger install base
- Incus's OCI/application-container support is newer than its system-container
  support and has seen less production mileage

We list these honestly because Incus doesn't need overselling - if your
workload fits, the experience speaks for itself.

## See Also

- [Getting Started](/getting-started) - install and run your first project
- [Compose Compatibility](/compose-compatibility) - what works and what does not
