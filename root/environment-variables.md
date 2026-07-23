---
date: 2026-07-05T01:24:44.707Z
dateCreated: 2026-07-05T01:03:10.392Z
description: null
editor: markdown
published: true
tags: []
title: Environment Variables
leafwiki_id: 20gXqlBDR
leafwiki_title: Environment Variables
leafwiki_created_at: "2026-07-05T03:53:59.566641541Z"
leafwiki_updated_at: "2026-07-10T03:03:00.048642311Z"
leafwiki_creator_id: vOmfrlBDg
leafwiki_last_author_id: vOmfrlBDg
---

# Environment Variables

incus-compose handles environment variables differently than docker-compose for security and reproducibility reasons.

## How It Works

### Default Behavior

By default, incus-compose loads environment variables from:

1. `.env` file in the compose file's directory
2. Files specified with `--env-file`

These `.env` files **can reference OS environment variables** for interpolation:

```env
# .env
DB_PASSWORD=secret123
HOME_DIR=${HOME}
CURRENT_USER=${USER}
```

Only variables explicitly defined in `.env` files are passed to your compose project. Your shell's environment (like `PATH`, `EDITOR`, etc.) is **not** automatically included.

### Why This Matters

- **Security**: Sensitive environment variables from your shell don't accidentally leak into containers
- **Reproducibility**: The same compose file behaves the same way on different machines
- **Explicitness**: You always know exactly which variables are available

## The `--os-env` / `-E` Flag

If you need full docker-compose compatibility, use the `--os-env` flag:

```bash
incus-compose --os-env up
incus-compose -E up
```

This includes all OS environment variables directly, matching docker-compose behavior.

## Examples

### Using .env files (recommended)

```env
# .env
DATABASE_URL=postgres://localhost/mydb
API_KEY=your-api-key
USER=${USER}
```

```yaml
# compose.yaml
services:
  app:
    environment:
      DATABASE_URL: ${DATABASE_URL}
      API_KEY: ${API_KEY}
      DEPLOYED_BY: ${USER}
```

```bash
incus-compose up
```

### Using --os-env for compatibility

```bash
export DATABASE_URL=postgres://localhost/mydb
incus-compose --os-env up
```

## Quick Reference

| Method     | Variables Available                         | Use Case                                    |
| ---------- | ------------------------------------------- | ------------------------------------------- |
| Default    | `.env` files only (can interpolate OS vars) | Production, CI/CD                           |
| `--os-env` | All OS environment variables                | Quick testing, docker-compose compatibility |

## CLI Configuration

Every global flag can be set via an environment variable. Flags given on the command line take precedence over environment variables.

Every command-specific flag can be set too, scoped per command as
`INCUS_COMPOSE_<COMMAND>_<FLAG>` - e.g. `--timeout` on `up` is
`INCUS_COMPOSE_UP_TIMEOUT`, `--timeout` on `down` is `INCUS_COMPOSE_DOWN_TIMEOUT`.
Each command gets its own variable even when the flag name is shared, so
setting one never leaks into another command. See [Command Flags](#command-flags)
for the full list, or run `incus-compose <command> --help` - every flag's
env var is shown inline as `[$VAR_NAME]`.

Four flags are the deliberate exception and have **no** environment variable,
because a forgotten shell variable would silently make every future
invocation destructive or a no-op instead of just changing cosmetic output:

| Flag              | Command | Why                                                |
| ----------------- | ------- | --------------------------------------------------- |
| `--recreate`       | `up`    | Would silently recreate containers on every `up`     |
| `--project`        | `down`  | Would silently delete the whole project              |
| `--volumes`        | `down`  | Would silently delete volumes                        |
| `--dry-run`        | `exec`  | Would silently no-op every `exec`, breaking scripts   |

### Project and Files

| Variable                          | Flag                        | Description                                                  |
| --------------------------------- | --------------------------- | ------------------------------------------------------------ |
| `INCUS_COMPOSE_FILE`              | `--file`, `-f`              | Compose configuration files (comma-separated for multiple)   |
| `INCUS_COMPOSE_PROJECT_NAME`      | `--project-name`, `-p`      | Project name                                                 |
| `INCUS_COMPOSE_PROJECT_DIRECTORY` | `--project-directory`, `-P` | Working directory                                            |
| `INCUS_COMPOSE_ENV_FILE`          | `--env-file`                | Alternative environment files (comma-separated for multiple) |
| `INCUS_COMPOSE_PROFILES`          | `--profile`                 | Profiles to enable (comma-separated for multiple)            |
| `INCUS_COMPOSE_OS_ENV`            | `--os-env`, `-E`            | Include OS environment variables for interpolation            |

### Incus Connection

| Variable                     | Flag             | Description                                                   |
| ---------------------------- | ---------------- | ------------------------------------------------------------- |
| `INCUS_REMOTE`               | `--remote`       | Incus remote name from CLI config (e.g., `local`, `myserver`) |
| `INCUS_COMPOSE_IMAGE_CACHE`  | `--image-cache`  | Incus project used as image cache (`INCUS_COMPOSE_IMAGE_CACHE`, default: `default`); set `""` to disable caching and pull straight into the project. Always disabled on Windows and macOS clients, regardless of this flag, see [CLI Reference](/cli-reference#global-options) |
| `INCUS_COMPOSE_STORAGE_POOL` | `--storage-pool` | Default storage pool (default: `detect`)                      |

### Display and Debugging

| Variable                | Flag        | Description                                                      |
| ----------------------- | ----------- | ---------------------------------------------------------------- |
| `INCUS_COMPOSE_ANSI`    | `--ansi`    | Control ANSI output: `never`, `always`, `auto` (default: `auto`) |
| `INCUS_COMPOSE_DEBUG`   | `--debug`   | Enable debug logging (`true`/`1`)                                |
| `INCUS_COMPOSE_WORKERS` | `--workers` | Number of concurrent workers (default: `4`)                      |
| `NO_COLOR`              | --          | Disable color output ([no-color.org](https://no-color.org/))     |

`--builder` and `--healthd-*` are command flags (`up`, `build`, `pull`, `healthd up`,
`healthd down`), not global ones - see [Command Flags](#command-flags) below for
their per-command variable names.

The ic-healthd daemon itself reads a further set of `INCUS_COMPOSE_HEALTHD_*`
variables (`_TOKEN`, `_PROJECTS`, `_OWN_PROJECT`, `_OWN_NAME`, `_DATA_DIR`,
`_SECRETS_DIR`, `_DEBUG`), which incus-compose injects into the sidecar. See [Running ic-healthd Directly](/healthd#running-ic-healthd-directly).

### Examples

```bash
# Use a configured Incus remote
export INCUS_REMOTE=myserver
incus-compose up

# Set project defaults in your shell profile
export INCUS_COMPOSE_FILE=compose.yaml,compose.prod.yaml
export INCUS_COMPOSE_PROJECT_NAME=myapp
incus-compose up

# Debug with extra workers
INCUS_COMPOSE_DEBUG=1 INCUS_COMPOSE_WORKERS=20 incus-compose up
```

## Command Flags

Every flag on every command below can be set via `INCUS_COMPOSE_<COMMAND>_<FLAG>`,
except the four listed under [CLI Configuration](#cli-configuration). Descriptions
are abbreviated; run `incus-compose <command> --help` for the full text and defaults.

### up / down / start / stop / restart

| Command   | Variable                             | Flag                 | Description                             |
| --------- | ------------------------------------- | --------------------- | ---------------------------------------- |
| `up`      | `INCUS_COMPOSE_UP_NO_START`           | `--no-start`           | Don't start containers after creating    |
| `up`      | `INCUS_COMPOSE_UP_TIMEOUT`            | `--timeout`            | Timeout for stopping/starting a service  |
| `up`      | `INCUS_COMPOSE_UP_DEPENDENCY_TIMEOUT` | `--dependency-timeout` | Max wait for `service_healthy` depends_on |
| `up`      | `INCUS_COMPOSE_UP_SCALE`              | `--scale`              | Scale SERVICE to NUM instances           |
| `up`      | `INCUS_COMPOSE_UP_PULL`               | `--pull`               | Pull policy                              |
| `up`      | `INCUS_COMPOSE_UP_BUILD`              | `--build`              | Build images before starting             |
| `up`      | `INCUS_COMPOSE_UP_BUILDER`            | `--builder`            | Preferred builder binary or path         |
| `up`      | `INCUS_COMPOSE_UP_NO_BUILD`           | `--no-build`           | Do not build images even if missing      |
| `up`      | `INCUS_COMPOSE_UP_NO_DEPS`            | `--no-deps`            | Don't start linked services              |
| `up`      | `INCUS_COMPOSE_UP_DETACH`             | `--detach`, `-d`       | Run containers in the background         |
| `up`      | `INCUS_COMPOSE_UP_NO_HEALTHD`         | `--no-healthd`         | Don't create the healthd sidecar         |
| `up`      | `INCUS_COMPOSE_UP_EXTERNAL_HEALTHD`   | `--external-healthd`   | Use healthd but don't create/look it up  |
| `up`      | `INCUS_COMPOSE_UP_HEALTHD_IMAGE`      | `--healthd-image`      | Healthd OCI image                        |
| `up`      | `INCUS_COMPOSE_UP_HEALTHD_BINARY`     | `--healthd-binary`     | Local ic-healthd binary path             |
| `up`      | `INCUS_COMPOSE_UP_HEALTHD_INCUS`      | `--healthd-incus`      | Incus API URL for the sidecar            |
| `up`      | `INCUS_COMPOSE_UP_HEALTHD_NETWORK`    | `--healthd-network`    | Network for the sidecar                  |
| `down`    | `INCUS_COMPOSE_DOWN_RMI`              | `--rmi`                | Remove images used by services           |
| `down`    | `INCUS_COMPOSE_DOWN_IMAGES`           | `--images`             | Remove known images from the project     |
| `down`    | `INCUS_COMPOSE_DOWN_TIMEOUT`          | `--timeout`            | Timeout for stopping                     |
| `down`    | `INCUS_COMPOSE_DOWN_NO_DEPS`          | `--no-deps`            | Don't stop linked services                |
| `down`    | `INCUS_COMPOSE_DOWN_EXTERNAL_HEALTHD` | `--external-healthd`   | Use healthd but don't look it up          |
| `down`    | `INCUS_COMPOSE_DOWN_NO_NETWORKS`      | `--no-networks`        | Don't touch networks                     |
| `start`   | `INCUS_COMPOSE_START_TIMEOUT`         | `--timeout`            | Timeout for starting                     |
| `start`   | `INCUS_COMPOSE_START_WITH_DEPS`       | `--with-deps`          | Also start linked services               |
| `stop`    | `INCUS_COMPOSE_STOP_TIMEOUT`          | `--timeout`            | Timeout for stopping                     |
| `stop`    | `INCUS_COMPOSE_STOP_WITH_DEPS`        | `--with-deps`          | Also stop linked services                 |
| `restart` | `INCUS_COMPOSE_RESTART_TIMEOUT`       | `--timeout`            | Timeout for stopping and starting        |
| `restart` | `INCUS_COMPOSE_RESTART_WITH_DEPS`     | `--with-deps`          | Also restart linked services              |

`up --recreate` and `down --project`/`--volumes` have no variable - see the
exceptions table above.

### build / pull

| Command | Variable                                  | Flag                    | Description                            |
| ------- | ------------------------------------------ | ------------------------ | ---------------------------------------- |
| `build` | `INCUS_COMPOSE_BUILD_NO_CACHE`             | `--no-cache`             | Do not use a cache when building         |
| `build` | `INCUS_COMPOSE_BUILD_PULL`                 | `--pull`                 | Pull policy                              |
| `build` | `INCUS_COMPOSE_BUILD_BUILDER`              | `--builder`              | Preferred builder binary or path         |
| `pull`  | `INCUS_COMPOSE_PULL_IGNORE_BUILDABLE`      | `--ignore-buildable`     | Ignore images that can be built          |
| `pull`  | `INCUS_COMPOSE_PULL_IGNORE_PULL_FAILURES`  | `--ignore-pull-failures` | Pull what it can, ignore failures        |
| `pull`  | `INCUS_COMPOSE_PULL_INCLUDE_DEPS`          | `--include-deps`         | Also pull linked services                |
| `pull`  | `INCUS_COMPOSE_PULL_POLICY`                | `--policy`               | Pull policy                              |
| `pull`  | `INCUS_COMPOSE_PULL_NO_HEALTHD`            | `--no-healthd`           | Don't pull the healthd sidecar           |
| `pull`  | `INCUS_COMPOSE_PULL_HEALTHD_IMAGE`         | `--healthd-image`        | Healthd OCI image                        |

### config

| Variable                          | Flag           | Description                             |
| ---------------------------------- | -------------- | ----------------------------------------- |
| `INCUS_COMPOSE_CONFIG_FORMAT`      | `--format`      | Output format: `yaml` or `json`           |
| `INCUS_COMPOSE_CONFIG_SERVICES`    | `--services`    | Print the service names, one per line     |
| `INCUS_COMPOSE_CONFIG_VOLUMES`     | `--volumes`     | Print the volume names, one per line      |
| `INCUS_COMPOSE_CONFIG_NETWORKS`    | `--networks`    | Print the network names, one per line     |
| `INCUS_COMPOSE_CONFIG_PROFILES`    | `--profiles`    | Print the profile names, one per line     |
| `INCUS_COMPOSE_CONFIG_QUIET`       | `--quiet`, `-q` | Only validate, don't print anything       |
| `INCUS_COMPOSE_CONFIG_IMAGES`      | `--images`      | Print the image names, one per line       |
| `INCUS_COMPOSE_CONFIG_ENVIRONMENT` | `--environment` | Print environment used for interpolation  |
| `INCUS_COMPOSE_CONFIG_VARIABLES`   | `--variables`   | Print model variables and default values  |
| `INCUS_COMPOSE_CONFIG_OUTPUT`      | `--output`, `-o`| Save to file (default: stdout)            |

### list / ps

| Command | Variable                       | Flag                | Description                              |
| ------- | ------------------------------- | -------------------- | ------------------------------------------ |
| `list`  | `INCUS_COMPOSE_LIST_FORMAT`     | `--format`           | Output format: `table`, `yaml` or `json`  |
| `list`  | `INCUS_COMPOSE_LIST_NO_HEALTHD` | `--no-healthd`       | Don't list the healthd sidecar             |
| `ps`    | `INCUS_COMPOSE_PS_ALL`          | `--all`, `-a`        | Show all containers, including stopped     |
| `ps`    | `INCUS_COMPOSE_PS_QUIET`        | `--quiet`, `-q`      | Only display Incus instance names          |
| `ps`    | `INCUS_COMPOSE_PS_SERVICES`     | `--services`         | Display services instead of instances      |
| `ps`    | `INCUS_COMPOSE_PS_FORMAT`       | `--format`           | Output format: `table` or `json`          |
| `ps`    | `INCUS_COMPOSE_PS_WITH_DEPS`    | `--with-deps`        | Also list linked services                  |

### logs / exec / self-update

| Command       | Variable                             | Flag              | Description                            |
| -------------- | ------------------------------------- | ------------------ | ----------------------------------------- |
| `logs`         | `INCUS_COMPOSE_LOGS_FOLLOW`           | `--follow`, `-f`   | Follow log output                        |
| `exec`         | `INCUS_COMPOSE_EXEC_DETACH`           | `--detach`, `-d`   | Run command in the background            |
| `exec`         | `INCUS_COMPOSE_EXEC_ENV`              | `--env`, `-e`      | Set environment variables (KEY=VALUE)    |
| `exec`         | `INCUS_COMPOSE_EXEC_INDEX`            | `--index`          | Replica index if service is scaled       |
| `exec`         | `INCUS_COMPOSE_EXEC_NO_TTY`           | `--no-tty`, `-T`   | Disable pseudo-TTY allocation            |
| `exec`         | `INCUS_COMPOSE_EXEC_PRIVILEGED`       | `--privileged`     | Accepted but not implemented             |
| `exec`         | `INCUS_COMPOSE_EXEC_USER`             | `--user`, `-u`     | Run the command as this user             |
| `exec`         | `INCUS_COMPOSE_EXEC_GROUP`            | `--group`, `-g`    | Run the command as this group            |
| `exec`         | `INCUS_COMPOSE_EXEC_WORKDIR`          | `--workdir`, `-w`  | Path to workdir directory                |
| `self-update`  | `INCUS_COMPOSE_SELF_UPDATE_DRAFT`     | `--draft`          | Also consider draft releases             |
| `self-update`  | `INCUS_COMPOSE_SELF_UPDATE_PRE_RELEASE` | `--pre-release`  | Also consider pre-releases               |

`exec --dry-run` has no variable - see the exceptions table above.

### healthd up / down / logs / restart

These are the `incus-compose healthd <subcommand>` management commands (see
[CLI Reference](/cli-reference#healthd)), distinct from `up`'s own `--healthd-*`
flags above.

| Command          | Variable                              | Flag        | Description                     |
| ----------------- | -------------------------------------- | ----------- | ---------------------------------- |
| `healthd up`      | `INCUS_COMPOSE_HEALTHD_UP_IMAGE`       | `--image`   | Healthd OCI image                |
| `healthd up`      | `INCUS_COMPOSE_HEALTHD_UP_BINARY`      | `--binary`  | Local ic-healthd binary path     |
| `healthd up`      | `INCUS_COMPOSE_HEALTHD_UP_INCUS`       | `--incus`   | Incus API URL for the sidecar    |
| `healthd up`      | `INCUS_COMPOSE_HEALTHD_UP_NETWORK`     | `--network` | Network for the sidecar          |
| `healthd up`      | `INCUS_COMPOSE_HEALTHD_UP_PULL`        | `--pull`    | Pull policy                      |
| `healthd up`      | `INCUS_COMPOSE_HEALTHD_UP_TIMEOUT`     | `--timeout` | Timeout for stopping             |
| `healthd down`    | `INCUS_COMPOSE_HEALTHD_DOWN_IMAGE`     | `--image`   | Healthd OCI image                |
| `healthd down`    | `INCUS_COMPOSE_HEALTHD_DOWN_TIMEOUT`   | `--timeout` | Timeout for stopping             |
| `healthd logs`    | `INCUS_COMPOSE_HEALTHD_LOGS_FOLLOW`    | `--follow`, `-f` | Follow log output            |
| `healthd restart` | `INCUS_COMPOSE_HEALTHD_RESTART_TIMEOUT` | `--timeout` | Timeout for stopping            |

## See Also

- [CLI Reference](/cli-reference) - command options and flags
- [Compose Compatibility](/compose-compatibility) - interpolation and env_file support
