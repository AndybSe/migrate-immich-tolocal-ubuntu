# Immich Migration: Hetzner VPS → MacBook Pro (Local Podman)

> **NOTE:** This guide was for a MacBook Pro destination. The actual destination is
> a local Ubuntu server at `192.168.88.16`. See `MIGRATION_TO_UBUNTU_SERVER.md`
> for the current migration plan. This file kept for reference only.

Migrates Immich from the Hetzner CPX42 VPS (204.168.139.34) to a local 2019
MacBook Pro 16" (Intel) with an external USB drive.

**Result:** Local-only access at http://localhost:2283, saves €31/month.

---

## Prerequisites

- macOS with Homebrew installed
- External USB drive (250GB+ free space)
- SSH access to VPS (root@204.168.139.34)

## Phase 1: Prepare the Mac

```bash
# Install dependencies
brew install podman podman-compose

# Initialize Podman machine (Intel Mac uses QEMU)
# IMPORTANT: Replace /Volumes/ImmichDrive with your actual USB drive path
podman machine init \
  --cpus 4 \
  --memory 3072 \
  --disk-size 20 \
  --volume /Volumes/ImmichDrive:/Volumes/ImmichDrive

podman machine start

# Create directory structure on USB drive
mkdir -p /Volumes/ImmichDrive/immich/{library,postgres,backups}
```

## Phase 2: Export Data from VPS

```bash
# SSH into VPS
ssh root@204.168.139.34

# Dump the database (runs as the immich user who owns the containers)
sudo -iu immich podman exec -t immich_postgres \
  pg_dumpall -c -U postgres > /opt/immich/backups/dump_final.sql

# Verify the dump is not empty
ls -lh /opt/immich/backups/dump_final.sql
head -20 /opt/immich/backups/dump_final.sql

# Exit VPS
exit
```

## Phase 3: Transfer Data to Mac

```bash
# Transfer database dump
scp root@204.168.139.34:/opt/immich/backups/dump_final.sql \
  /Volumes/ImmichDrive/immich/backups/

# Transfer photo library (~250GB — run in tmux, takes 6-8 hours)
# rsync is resumable if interrupted
rsync -avz --progress \
  root@204.168.139.34:/opt/immich/library/ \
  /Volumes/ImmichDrive/immich/library/
```

## Phase 4: Deploy Immich Locally

```bash
# Copy config files from this repo to USB drive
cp podman-compose.yml /Volumes/ImmichDrive/immich/
cp .env /Volumes/ImmichDrive/immich/

# Create the Podman network
podman network create immich_network

# Start database only (need it running for restore)
cd /Volumes/ImmichDrive/immich
podman-compose up -d database

# Wait for PostgreSQL to initialize
sleep 15

# Verify database is accepting connections
podman exec immich_postgres pg_isready -U postgres

# Restore the SQL dump
# NOTE: The fresh .env has a new password. The dump includes role definitions
# from the old server, but POSTGRES_PASSWORD in .env takes precedence for the
# postgres superuser on container init. If you see auth errors after restore,
# run: podman exec immich_postgres psql -U postgres -c "ALTER USER postgres PASSWORD 'zrr4cq7Y+PLAYOE458ZlhkfGugEg9zm0';"
cat backups/dump_final.sql | podman exec -i immich_postgres psql -U postgres

# Start all remaining services
podman-compose up -d
```

## Phase 5: Verify

### Automated checks

```bash
# All 4 containers running?
podman ps --format "table {{.Names}}\t{{.Status}}"

# Server responding?
curl -s http://localhost:2283/api/server/ping
# Expected: {"res":"pong"}
```

### Manual checks

1. Open http://localhost:2283
2. Log in with your existing credentials
3. Spot-check 5-10 albums for correct photo count
4. Verify face recognition data carried over
5. Test search functionality
6. Upload a test photo to verify writes work

## Phase 6: Set Up Convenience Scripts

```bash
# Option A: Symlink scripts to ~/bin
mkdir -p ~/bin
ln -s /Volumes/ImmichDrive/immich/immich-start.sh ~/bin/immich-start
ln -s /Volumes/ImmichDrive/immich/immich-stop.sh ~/bin/immich-stop

# Option B: Shell aliases in ~/.zshrc
echo 'alias immich-start="/Volumes/ImmichDrive/immich/immich-start.sh"' >> ~/.zshrc
echo 'alias immich-stop="/Volumes/ImmichDrive/immich/immich-stop.sh"' >> ~/.zshrc

# Daily usage:
immich-start          # Start everything, opens browser
immich-stop           # Stop containers, safe to eject
immich-stop --full    # Also stop Podman VM to free RAM
```

## Phase 7: Decommission VPS (after 1 week)

1. Take a final VPS snapshot in Hetzner console (insurance)
2. Keep VPS running for 1 week as safety net
3. Delete the Hetzner server via console
4. Cancel backup addon
5. Update `docs/guides/IMMICH_DEPLOYMENT.MD` to reflect local setup

## Troubleshooting

**Container won't start / volume errors:**
Verify the USB drive path is mounted and the Podman machine has the volume:
```bash
podman machine inspect | grep -A5 Mounts
```
If the volume mount is missing, recreate the machine with `--volume`.

**Database auth errors after restore:**
The dump may contain old role passwords. Reset after restore:
```bash
podman exec immich_postgres psql -U postgres \
  -c "ALTER USER postgres PASSWORD 'zrr4cq7Y+PLAYOE458ZlhkfGugEg9zm0';"
```

**Slow performance (especially ML/search):**
With 16GB RAM and 3GB allocated to the Podman VM, ML indexing will be slow.
This is expected for CPU-only inference on Intel. Initial re-indexing may take
hours — let it run overnight. Day-to-day use (browsing, uploading) is fine.

**PostgreSQL slow on USB drive:**
If DB queries feel sluggish, consider keeping postgres data on the internal SSD
instead. Update `DB_DATA_LOCATION` in `.env` to a local path (e.g.,
`/Volumes/ImmichDrive/immich/postgres` → `~/immich-postgres`), and add that
path to the Podman machine volume mounts.
