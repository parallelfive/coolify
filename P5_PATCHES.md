# Parallel Five — Custom Coolify Image

This fork exists to maintain a patched Coolify image for the Parallel Five homelab (MacBook Pro running Coolify behind a Cloudflare Tunnel).

## Why

Coolify doesn't officially support macOS or Cloudflare Tunnel deployments. We've hit several bugs that require source-level patches. Rather than volume-mounting patched files and re-patching after every update, we maintain a thin Docker image that extends the official Coolify image with our fixes.

## Image

```
ghcr.io/parallelfive/coolify:latest
```

Built automatically by GitHub Actions on push to `p5/patched` and weekly (to pick up base image updates).

## Branch Layout

| Branch | Purpose |
|---|---|
| `v4.x` | Upstream mirror (don't commit here) |
| `next` | Upstream dev branch |
| `p5/patched` | **Our branch** — patches on top of `v4.x`, builds the custom image |
| `fix/*` | Upstream PR branches |

## Active Patches

### 1. PrivateKey.php — SSH key overwrite fix
- **File**: `app/Models/PrivateKey.php`
- **Problem**: `storeInFileSystem()` chmods keys to 0600. Flysystem's `put()` returns false on overwrite because it can't set visibility on a 0600 file. Every deploy after the first fails with "Failed to write SSH key to filesystem."
- **Fix**: Delete existing file before `put()`.
- **Upstream PR**: [coollabsio/coolify#9491](https://github.com/coollabsio/coolify/pull/9491)
- **Remove when**: PR is merged and included in a release.

### 2. StartSentinel.php — macOS bind mount fix
- **File**: `app/Actions/Server/StartSentinel.php`
- **Problem**: Sentinel uses a bind mount to `/data/coolify/sentinel` which fails on macOS due to chmod permission errors (Docker Desktop UID mapping).
- **Fix**: Uses a Docker volume (`coolify-sentinel-data`) instead of a bind mount, and skips chown/chmod.
- **Remove when**: Coolify adds native macOS support (unlikely soon).

### 3. default.conf — Nginx config for Cloudflare Tunnel
- **File**: `patches/default.conf`
- **Target**: `/etc/nginx/conf.d/default.conf`
- **Problem**: Default nginx config doesn't proxy websocket paths needed for terminal and realtime features through a Cloudflare Tunnel.
- **Fix**: Custom server block that includes the terminal proxy config.
- **Remove when**: Coolify adds Cloudflare Tunnel support (unlikely soon).

### 4. terminal-proxy.conf — Websocket proxy
- **File**: `patches/terminal-proxy.conf`
- **Target**: `/etc/nginx/site-opts.d/terminal.conf`
- **Problem**: Terminal and realtime websockets don't work through Cloudflare Tunnel without explicit nginx proxying.
- **Fix**: Proxies `/terminal/ws` → `coolify-realtime:6002` and `/app/` → `coolify-realtime:6001`.
- **Remove when**: Same as above.

### 5. DatabaseBackupJob.php — POSIX size check for macOS hosts
- **File**: `app/Jobs/DatabaseBackupJob.php` (method `calculate_size`)
- **Problem**: `calculate_size()` runs `du -b $file | cut -f1`, but the `-b` byte-count flag is a GNU coreutils extension not available on macOS/BSD `du`. On macOS hosts, `du -b` errors out, Coolify reads empty output as size=0, throws `Local backup file is empty or was not created` — even when the pg_dump file itself is valid and non-zero. Silent-failure-like-nuisance: backups run successfully but get marked failed + skipped for S3 upload.
- **Fix**: Replace with `wc -c < $file | tr -d ' '` — POSIX, works identically on GNU coreutils and BSD. `tr -d ' '` strips BSD wc's leading whitespace so the integer parses cleanly.
- **Remove when**: Upstream adopts a portable size check (file a PR when we have a moment).

## How to Update

### When Coolify releases a new version

1. Sync the fork:
   ```bash
   git fetch upstream
   git checkout p5/patched
   git rebase upstream/v4.x
   ```
2. Resolve any conflicts in patched files.
3. Push — the Action rebuilds automatically.
4. On the MacBook: `docker compose pull coolify && docker compose up -d coolify`

### Adding a new patch

1. Apply the change to the relevant file on `p5/patched`.
2. If it's a non-source file (nginx config, etc.), put it in `patches/` and add a `COPY` line to `Dockerfile.p5`.
3. Push — Action rebuilds.
4. Document the patch in this file.

### When an upstream PR is merged

Remove the corresponding `COPY` line from `Dockerfile.p5`, revert the patched file to upstream, and update this doc.

## Infrastructure

- **MacBook Pro**: `ssh thchristmas` (Tailscale)
- **Compose files**: `/data/coolify/source/docker-compose.yml` + `docker-compose.prod.yml`
- **Image config**: `docker-compose.prod.yml` → `image: ghcr.io/parallelfive/coolify:latest`
- **Volume mounts**: Still in `docker-compose.prod.yml` as a fallback (belt and suspenders)
