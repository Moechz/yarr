# yarr-for-tos

> TerraMaster TOS 7 App Center — **Docker application** package for
> [Yarr](https://github.com/nkanaev/yarr).

## What this is

A four-file TOS Docker application archive (`yarrdocker.tar.gz`) that installs
Yarr on TOS 7 through the App Center:

- `config.ini` — TOS application metadata (`application_type: docker`, id `yarrdocker`)
- `yarrdocker.lang` — 23-language superset (TOS requires 14)
- `yarrdocker.svg` — application icon (625 bytes, 8 SVG elements)
- `docker-compose.yml` — one service, image `nkanaev/yarr:v2.9`

## How it works

| | |
|---|---|
| Container image | `nkanaev/yarr:v2.9` (Docker Hub, used unmodified) |
| Open | `http://<NAS-IP>:8161/` (container port 7070) |
| Runs as | non-root, `user: "1000:1000"` |
| Persistent data | `/data` → `/Volume1/DockerAppData/yarrdocker/data` |
| Logs | `docker logs yarrdocker` (stdout/stderr) |

`/data` (mounted from `DockerAppData/yarrdocker/data`) holds `yarr.db` with feeds and read state.

## Design highlights

- **Non-root.** `user: "1000:1000"`; verified on real hardware (the container reports uid 1000).
- **Fixed image tag, never `:latest`**, Docker Hub only, no `privileged`, no `network_mode: host`, and no Docker socket, no `cap_add`.
- **Health check** probing the service on container port 7070.
- **Deterministic build**: GNU tar with uid/gid 0 and zeroed timestamps, gzip without a timestamp, plus the review-standards self-check above.
- **Health check without HTTP tools.** The image ships no curl/wget/nc, so the check greps the listening port in `/proc/net/tcp6` (7070 = 0x1B9E).
- **Runs as uid 1000.** One SQLite file on the mounted volume is all the state it has.

## Privacy

This application runs entirely on your NAS. It does not include analytics, telemetry or advertising SDKs, and the packaging contains no code that sends data to the packager. Data is only exchanged with the services you configure yourself (for example media servers, notification endpoints or model APIs), and only from the container. Everything the application stores lives in the application data folder described above; removing the app with its data deletes it.

## Third-party components

The package runs the upstream image `nkanaev/yarr:v2.9` unmodified. Its components and their licences are those of the upstream project (https://github.com/nkanaev/yarr), which is distributed under **MIT**. No upstream source code is redistributed in this repository; only the four packaging files are original work.

## Runtime file manifest

Everything the application writes on the NAS lives inside its own data
directory; the container-internal `/tmp` is discarded with the container. The
table lists what is created and how it behaves. Nothing else is written.

| Path | Purpose | Created when | Growth bound | Lifecycle |
|---|---|---|---|---|
| `/Volume1/DockerAppData/yarrdocker/data` | Yarr application data (everything the container writes outside /tmp) | on first start / on demand | bounded by your own data | persistent; kept on uninstall unless you ask to delete it |

## Build

```bash
scripts/build.sh                  # -> out/yarrdocker.tar.gz + .sha256
scripts/build.sh 2.9-2   # packaging iteration bump
scripts/build.sh 2.9-1 aarch64
```

The build stages the four files, substitutes the version/platform placeholders,
removes CRLF/BOM/AppleDouble, then runs
`tools/verify.py`, a review-standards self-check (archive layout, JSON validity,
required fields, 14+ language coverage, version consistency, Docker-Hub-only
images, fixed tag, reserved/recommended host ports, non-root user, per-service
healthcheck/TZ/restart, data path below the application data root, never the
`/Volume*` wildcard, comment-free sequence blocks, `x-app-meta` position, no
literal secrets and the SVG size/element limits).

## Local verification

```bash
./TOSAppSelfTestingTool -c out/yarrdocker.tar.gz   # official spec check
bash test/install-and-verify.sh                  # on-device E2E (needs root)
```

> Docker applications cannot be side-loaded through the App Center's manual
> install page (it accepts `.deb` only); the official self-testing tool's `-i`
> mode or `docker compose` are the only ways to install one for testing.

## Store submission checklist

1. Public GitHub repo (code + README only).
2. Release **tag `v2.9-1`**, assets `yarrdocker.tar.gz` +
   `yarrdocker.tar.gz.sha256` (Docker asset naming: `<app_id>.tar.gz`, **no
   platform suffix**).
3. Developer platform → Add Application: ID `yarrdocker`, package type Docker,
   repository URL, architecture `x86_64`.
4. Version Management → Add Version `2.9-1` → automated validation → review.

## License / branding

Yarr is distributed by its upstream authors under **MIT**. The container image is the official image from Docker Hub (`nkanaev/yarr:v2.9`) and is used unmodified; no upstream source code is redistributed here. This packaging is an independent community submission by Moechz and is not affiliated with the upstream project or TerraMaster.

## Limitations

- No authentication; use a reverse proxy if it must be reachable from outside the LAN.
- The image has no shell utilities beyond `sh`/`grep`.
- The UI is plain HTTP.

---

Packaged for TOS 7 by Moechz. Upstream image used unmodified from Docker Hub.
