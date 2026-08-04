---
date: 2026-07-05T01:27:55.757Z
dateCreated: 2026-07-05T01:03:28.732Z
description: null
editor: markdown
published: true
tags: []
title: Testing Guide
leafwiki_id: 9ykuqlBDR
leafwiki_title: Testing Guide
leafwiki_created_at: "2026-07-05T03:54:00.828566786Z"
leafwiki_updated_at: "2026-07-08T02:52:44.530152914Z"
leafwiki_creator_id: vOmfrlBDg
leafwiki_last_author_id: vOmfrlBDg
---

# Testing Guide

This guide covers testing patterns, fixtures, and best practices for incus-compose.

## Prerequisites

- **gotestsum** - required to run tests via [just](https://github.com/casey/just/releases); install with `go install gotest.tools/gotestsum@latest`
- **just** - must be a recent version; the one shipped with Debian Trixie is too old and will not work
- **jq** - the purge commands require `jq` to be installed.

## Running Tests

Use `just --list` to see all available commands. Below is the complete reference:

**Run with**:

```bash
just test
```

### Image Cache

Tests use a dedicated cache project (`incus-compose-tests-cache`) separate from the CLI's image cache (the `default` project unless `--image-cache` is set). This keeps test images isolated and avoids polluting the user's cache.

The test cache is configured via `ClientProvideConnection` in test setup, pointing to a test-specific project.

### Environment Setup

The nested Incus environment is configured via `.env` file:

- `INCUS_REMOTE` - The remote to use.
- `TEST_PROCS` (default 2) - number of tests to run in parallel.
- `INCUS_COMPOSE_WORKERS` (default 4) - number of resources to create in parallel per test.

There's also `just test-e2e` which includes slow (long-running) tests.

### Test Commands

| Command                                         | Description                                                    |
| ----------------------------------------------- | -------------------------------------------------------------- |
| `just test`                                     | Run all tests against nested Incus (preferred also runs in CI) |
| `just test ./client/...`                        | Run tests for specific package                                 |
| `just test -v -run TestName`                    | Run specific test with verbose output                          |
| `just test-local`                               | Run unit tests only (no Incus connection required)             |
| `just test-e2e`                                 | Run E2E tests that take long to run                            |
| `just update-snapshots`                         | Update all snapshot test files                                 |
| `just update-snapshots ./cmd/incus-compose/...` | Update snapshots for specific package                          |
| `just update-e2e-snapshots`                     | Update snapshot for E2E test files                             |

### Development Commands

| Command                 | Description                                  |
| ----------------------- | -------------------------------------------- |
| `just build`            | Build the binary                             |
| `just run <args>`       | Run incus-compose via `go run` (uses `.env`) |
| `just run-local <args>` | Run against local Incus (ignores `.env`)     |
| `just incus <args>`     | Run commands in the nested Incus container   |

### Code Quality

| Command           | Description                              |
| ----------------- | ---------------------------------------- |
| `just lint`       | Lint all files with golangci-lint        |
| `just fix`        | Fix lint issues with golangci-lint       |
| `just pre-commit` | Run before committing (tidy, lint, test) |

### Setup & Maintenance

| Command            | Description                         |
| ------------------ | ----------------------------------- |
| `just dev-install` | Create nested Incus dev environment |

## Test Organization

Tests live alongside the code they test:

```
client/
  ├── client.go
  ├── client_test.go      # Tests for client.go
  ├── resource_image.go
  └── resource_image_test.go   # Tests for resource_image.go
project/
  ├── project.go
  └── project_test.go     # Tests for project.go
```

## Unit, integration and E2E

Tests are not split by directory or build tag. Which tier a test belongs to is
decided by the skip helper it calls on its first line:

| Tier | Guard | Needs Incus | Runs with |
| --- | --- | --- | --- |
| unit | none | no | every command, including `just test-local` |
| integration | `skipLocal(t)` | yes | `just test` |
| E2E | `skipE2E(t)` | yes | `just test-e2e` |

- **Unit** tests exercise pure logic - name parsing and sanitizing, config
  translation, argument building, `buildArgs`, snapshot rendering. No guard, so
  they run everywhere and must stay fast.
- **Integration** tests call `skipLocal(t)` and drive a real nested Incus. Most
  resource tests live here: they create a throwaway project, act on it, and let
  `t.Cleanup` tear it down. `INCUS_COMPOSE_TEST_LOCAL=1` (set by `just
  test-local`) skips them, which is why a green `just test-local` proves much
  less than a green `just test`.
- **E2E** tests call `skipE2E(t)` and are the slow, full-CLI ones. They are
  skipped unless `INCUS_COMPOSE_TEST_E2E=1` (set by `just test-e2e`), so they
  stay out of the normal loop.

There is no mocking of `incus.InstanceServer`. Anything that needs Incus talks
to the real nested one; that is the point of the integration tier.

**Examples**: `client/resource_image_test.go` mixes all three - parsing tests
with no guard, ensure/lock tests behind `skipLocal`.

**Run with**:

```bash
just test-local   # unit only
just test         # unit + integration
just test-e2e     # unit + integration + E2E
```

### The one mock

There is a single mock, `mockResource` in `client/resource_test.go`. It exists
to test ordering logic (`groupByPriority`) without touching a server, and it
implements `Resource` only:

```go
func newMockResource(name string, kind Kind, priority int, ensured bool) *mockResource
```

Use it rather than writing another; anything needing more than a name, kind and
priority belongs in the integration tier against real Incus.

## Prove the test red before you trust it green

A test written against a fix you just made passes for two possible reasons: the
fix works, or the test never checked anything. Those are indistinguishable until
you make it fail.

So before a fix is done, break it back and watch the test go red:

```bash
# disable the fix (an `if false &&` on the guard is enough), then:
just test ./client/ -run TestTheThing -count=1
```

Two things this catches regularly:

- **Assertions that cannot fail.** A `require.Error` passes on *any* error,
  including "builder not found" when you meant to prove "the builder ran". Assert
  on something only the real path produces.
- **Setups that never reproduce the condition.** A concurrency test whose workers
  resolve to different names never contends, and passes whether or not the fix
  exists. If the test still passes with the fix disabled, the test is wrong, not
  the fix.

Always pass `-count=1` when re-running: Go caches successful results and a cached
`0.000s` "pass" tells you nothing about the code you just changed. For anything
concurrent, use `-count=5` or more - a race that reproduces one run in three will
otherwise look fixed.

## Test the failures too

Green-path coverage only shows the feature works when everything is available.
Most of what users hit is the other half, and error behaviour is exactly what
regresses silently:

- **Every guard needs a test.** `pull never` with nothing stored, `--no-build`
  with a missing image, no source and no cache configured. Each guard is a
  branch, and an untested branch is a branch that stops working.
- **Assert the error, not just that one happened.** `require.ErrorIs(err,
  ErrNotFound)` pins the contract; `require.Error(err)` accepts a typo in a
  URL. Sentinel errors exist so callers can branch on them - test them the way a
  caller would.
- **Assert what did *not* happen.** Often the real contract is an absence: the
  cache was not repopulated, the builder was not invoked, the other lock was not
  released. A pointing-at-nothing build context or a nulled-out source turns
  "it didn't go there" into something you can assert.

## Test Fixtures

Located in `test/fixtures/`. Each fixture is a minimal compose scenario.

### Available Fixtures

- `simple-nginx/` - Simplest case
- `wordpress/` - Multi-service with volumes
- `with_profiles/` - Profile testing
- `with_env/` - Environment variable testing
- `with-secrets/` - Secrets management testing
- `with-restart/` - Restart policies testing

### Fixture Guidelines

**Snapshot portability**: Normalize absolute paths before snapshotting:

```go
output = strings.ReplaceAll(output, fixturePath, "$FIXTURE_PATH")
```

**Self-contained fixtures**: Define env vars like `$USER` or `$HOME` in `.env` to avoid OS dependencies:

```env
USER=testuser
HOME=/home/testuser
```

**Pure YAML**: Compose files should be pure YAML without comments:

```yaml
services:
  web:
    image: images:alpine/edge
    ports:
      - "8080:80"
```

### Snapshot Tests

Snapshots live in `test/snapshots/` and are named by test function and case.

**Update snapshots**:

```bash
just update-snapshots
```

**Snapshot naming**: `TestFunctionName_TestCase.yaml`

### Common Workflows

```bash
# Run a single test verbosely
just test -v -run TestInstanceSecretSuite

# Run tests for a specific package
just test ./client/...

# Quick validation before commit
just pre-commit

# Test a compose file
just run -f test/fixtures/simple-nginx/compose.yaml config
```

## Best Practices

1. **Test isolation** - Each test gets fresh resources via `SetupTest()`
2. **Error aggregation** - Use `errors.Join()` for batch operation errors
3. **Priority testing** - Verify creation/deletion order respects priorities
4. **Mock consistency** - Mocks should behave like real resources
5. **Fixture reuse** - Share fixtures across tests but keep them minimal
6. **Snapshot hygiene** - Review snapshot diffs carefully during updates

## See Also

- [Contributing](https://github.com/lxc/incus-compose/blob/main/CONTRIBUTING.md) - coding, style, and workflow rules
- [Architecture](/architecture) - the design these tests exercise
- [Client Package](/architecture/client) - Stack and resource internals
