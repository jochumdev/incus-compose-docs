---
tags: []
leafwiki_id: suel3lBvR
leafwiki_title: Docs
leafwiki_created_at: "2026-07-05T03:53:58.9035939Z"
leafwiki_updated_at: "2026-07-05T04:14:46.479994409Z"
leafwiki_creator_id: vOmfrlBDg
leafwiki_last_author_id: vOmfrlBDg
---
# Docs

## Using incus-compose

- [Getting Started](/docs/getting-started) - Install and run your first compose project
- [Terminology](/docs/terminology) - Compose vs Incus vs incus-compose terms (service vs instance, ...)
- [CLI Reference](/docs/cli-reference) - All commands and options
- [Compose Compatibility](/docs/compose-compatibility) - Supported features and differences
- [Builds](/docs/builds) - Build service images from Compose `build:` definitions
- [Health Checking](/docs/healthd) - Healthchecks, restart policies, and `service_healthy` dependencies via the ic-healthd sidecar (a core component; includes [debugging](/docs/healthd#debugging-ic-healthd))
- [Environment Variables](/docs/environment-variables) - How env vars and interpolation work
- [Why Incus?](/docs/why-incus) - Benefits over Docker

## Contributing and Internals

- [Contributing](https://github.com/lxc/incus-compose/blob/main/CONTRIBUTING.md) - Coding, style, and workflow rules
- [Architecture](/docs/architecture) - Resource-first design, layers, x-incus extensions
- [Client Package](/docs/architecture/client) - Resources, Stack, WorkerPool, hooks
- [Testing](/docs/architecture/testing) - just commands, test patterns, fixtures
- [GitHub Actions Runner](/docs/github-runner) - Set up a self-hosted runner for the test suite
