---
tags: []
leafwiki_id: 8sctjGYvR
leafwiki_title: Windows
leafwiki_created_at: "2026-07-09T00:41:29.080592548Z"
leafwiki_updated_at: "2026-07-09T00:48:20.173622057Z"
leafwiki_creator_id: vOmfrlBDg
leafwiki_last_author_id: vOmfrlBDg
---
# Installing on Windows

Incus itself is a Linux daemon - it does not run on Windows. On Windows you run
the `incus` client and `incus-compose` as **clients** that drive a remote Incus
server over HTTPS. No Docker and no WSL required.

## Prerequisites

- Windows 10/11, `x86_64` or `arm64`.
- A reachable Incus server (Linux) with `core.https_address` set. This is required
  even beyond remote access - see
  [Getting Started - Incus must listen on the network](/getting-started#incus-must-listen-on-the-network-required).
- Admin access to that server to trust your client certificate.

## 1. Create a bin directory and add it to your PATH

Create `%LOCALAPPDATA%\bin` and add it to your user `PATH` (Settings -> "Edit
environment variables for your account" -> `Path` -> New). Open a **new** terminal
afterwards so the change takes effect.

![environment.png](/assets/8sctjGYvR/environment.png)

## 2. Install the incus client

Download `bin.windows.incus.{x86_64|arm64}.exe` from the Incus
[Releases](https://github.com/lxc/incus/releases) page into `%LOCALAPPDATA%\bin`
and rename it to `incus.exe`.

## 3. Install incus-compose

Download `incus-compose_1.0.0-rc.3_windows_{amd64|arm64}.tar.gz` from the
incus-compose [Releases](https://github.com/lxc/incus-compose/releases) page,
extract it, and copy `incus-compose.exe` into `%LOCALAPPDATA%\bin`.

![binaries.png](/assets/8sctjGYvR/binaries.png)

Verify both are on your PATH in a new PowerShell terminal:

```powershell
incus version
incus-compose version
```

## 4. Generate a client certificate

Make sure your clock is correct (TLS verification fails on a skewed clock), then
generate a client certificate:

```powershell
incus remote generate-certificate
```

Print the certificate so you can copy it as text:

```powershell
incus remote get-client-certificate
```

![get-client-certificate.png](/assets/8sctjGYvR/get-client-certificate.png)


## 5. Trust the certificate on the server

On the **Incus server**, save the copied certificate to a file and add it to the
trust store:

```bash
# paste the certificate into hostname.crt, then:
incus config trust add-certificate ./hostname.crt
```

## 6. Add the server as a remote

Back on Windows, add the server and make it your default remote:

```powershell
incus remote add <servername> <server-ip>
incus remote switch <servername>
```

Test the connection:

```powershell
incus list --all-projects
```

![remote-list.png](/assets/8sctjGYvR/remote-list.png)


## 7. Add OCI image remotes (optional)

To pull container images, register the registries as Incus remotes:

```powershell
incus remote add --protocol oci docker.io https://docker.io
incus remote add --protocol oci ghcr.io https://ghcr.io
```

## 8. incus-compose in action

![immich-up.png](/assets/8sctjGYvR/immich-up.png)
![immich-exec.png](/assets/8sctjGYvR/immich-exec.png)
![immich-down.png](/assets/8sctjGYvR/immich-down.png)


## Notes and limitations

- **Remote-only.** Because you always talk to a remote server over HTTPS,
  **bind mounts are not supported** - use named volumes instead. Health checks
  work automatically over HTTPS. See
  [Local vs Remote Incus](/compose-compatibility#local-vs-remote-incus).
- **Builds** need a local `podman` or `docker`. Pull prebuilt images where you
  can; see [Builds](/builds).

Have fun with incus and incus-compose on Windows!

## See Also

- [Getting Started](/getting-started)
- [CLI Reference](/cli-reference)
- [Compose Compatibility](/compose-compatibility)
