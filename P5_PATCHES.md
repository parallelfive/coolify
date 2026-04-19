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

### 6. CheckUpdates.php — skip OS-patch-check on macOS
- **File**: `app/Actions/Server/CheckUpdates.php` (method `handle`)
- **Problem**: `CheckUpdates` runs `cat /etc/os-release` to sniff distro + package manager. macOS has no `/etc/os-release`, so the outer catch returns `['error' => 'cat: /etc/os-release: No such file or directory']`. `ServerPatchCheckJob` treats that as a failure and fires the `Server patch check failed` notification every cycle — one email (or Slack ping) per 10-ish minutes forever.
- **Fix**: Before the Linux path, `instant_remote_process(['uname -s'], $server, throwError: false)`; if it's `Darwin`, return a clean zero-updates result (no `error` key) so the job exits silently without notifying.
- **Remove when**: Upstream handles non-Linux servers gracefully (unlikely — macOS isn't a supported target).

### 7. pr-quality.yaml — gate to upstream repo only
- **File**: `.github/workflows/pr-quality.yaml`
- **Problem**: Upstream's `peakoss/anti-slop` action enforces `allowed-target-branches: "next"` and has `close-pr: true`. PRs on this fork target `p5/patched`, so the action auto-closes every PR seconds after open/reopen.
- **Fix**: Job-level `if: github.repository == 'coollabsio/coolify'` — skip on our fork; upstream runs unchanged.
- **Remove when**: N/A (permanent fork guard).

### 8. coolify-staging-build.yml — skip p5/** branches
- **File**: `.github/workflows/coolify-staging-build.yml`
- **Problem**: Upstream's staging build fires on push to every branch except `v4.x/v3.x/*v5.x*`. Our `p5/*` branches triggered it and failed trying to push to `coollabsio/coolify` without creds.
- **Fix**: Add `p5/**` to the `branches-ignore` list.
- **Remove when**: N/A (permanent fork guard).

## How to Update

### ⚠️ Do NOT click "Sync fork"

The GitHub UI shows `N commits behind coollabsio/coolify:v4.x` on `p5/patched`. **That's cosmetic — don't hit the Sync button.** Syncing would:

1. Merge 90+ upstream commits into `p5/patched` in one shot.
2. Conflict on every patched file (upstream may have changed the surrounding code).
3. Pull in non-code churn (changelog, templates, etc.) we don't care about.

Our runtime is always upstream-latest anyway: `Dockerfile.p5` starts `FROM ghcr.io/coollabsio/coolify:latest`. The patched PHP files in this branch only need to be *current enough* to cleanly overlay onto whatever `:latest` contains. If one patch goes stale (security fix landed upstream in the same file, new behavior we'd clobber), do a **targeted rebase** of just that file — not a full-branch sync.

### Targeted rebase when a patch goes stale

```bash
git fetch upstream
git checkout p5/patched -b p5/sync-upstream-YYYY-MM
# For each stale patched file:
git show upstream/v4.x:app/Models/PrivateKey.php > app/Models/PrivateKey.php
# Re-apply our patch by hand (tracked in Active Patches above)
# Verify: diff <(git show upstream/v4.x:<file>) <file>  should read as our minimal delta
git commit -am "sync: rebase P5 patches onto upstream/v4.x (YYYY-MM)"
git push -u origin p5/sync-upstream-YYYY-MM
gh pr create -R parallelfive/coolify --base p5/patched --head p5/sync-upstream-YYYY-MM
```

After merge, CI rebuilds `ghcr.io/parallelfive/coolify:latest`, then on the MacBook:

```bash
ssh thchristmas 'cd /data/coolify/source && sudo docker compose pull coolify && sudo docker compose up -d coolify'
```

Reference run: PR #3 (2026-04-19) — 4 files touched, minimal diff surface per file. See commit `236cafb25` for the pattern.

### When to do it

Check periodically (monthly-ish, or when a Coolify security advisory lands):

```bash
git fetch upstream
for f in app/Models/PrivateKey.php app/Actions/Server/StartSentinel.php app/Jobs/DatabaseBackupJob.php app/Actions/Server/CheckUpdates.php; do
  echo "--- $f ---"
  git log upstream/v4.x --oneline --since=4.weeks -- "$f"
done
```

If a patched file shows any upstream commits, start a sync branch. If nothing shows, you're current.

### Adding a new patch

1. Apply the change to the relevant file on a `p5/patch-N-*` branch off `p5/patched`.
2. If it's a non-source file (nginx config, etc.), put it in `patches/` and add a `COPY` line to `Dockerfile.p5`.
3. For PHP patches, add the `COPY` line too.
4. PR into `p5/patched`. CI rebuilds.
5. Document the patch in this file.

### When an upstream PR supersedes one of ours

Remove the corresponding `COPY` line from `Dockerfile.p5`, delete the patched file, and update this doc. The next image build will use pure upstream for that file.

## Infrastructure

- **MacBook Pro**: `ssh thchristmas` (Tailscale)
- **Compose files**: `/data/coolify/source/docker-compose.yml` + `docker-compose.prod.yml`
- **Image config**: `docker-compose.prod.yml` → `image: ghcr.io/parallelfive/coolify:latest`
- **Volume mounts**: Still in `docker-compose.prod.yml` as a fallback (belt and suspenders)
