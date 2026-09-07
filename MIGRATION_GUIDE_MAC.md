# Immich Migration Guide: VPS → Mac

> **NOTE:** This guide was for a MacBook destination. The actual destination is
> a local Ubuntu server at `192.168.88.16`. See `MIGRATION_TO_UBUNTU_SERVER.md`
> for the current migration plan. This file kept for reference only.

All commands below run on your **Mac** unless noted otherwise.

## Prerequisites

- External USB drive "ImmichDrive" plugged in (mounted at `/Volumes/ImmichDrive`)
- SSH access to VPS: `ssh root@204.168.139.34`

---

## Step 1: Install Podman

```bash
brew install podman podman-compose
```

## Step 2: Create directory structure on USB drive

```bash
mkdir -p /Volumes/ImmichDrive/immich/{library,postgres,backups}
```

## Step 3: Copy config files to USB drive

If you cloned the Infrastructure repo on the Mac:

```bash
cp infrastructure/mac/immich/podman-compose.yml /Volumes/ImmichDrive/immich/
cp infrastructure/mac/immich/.env.example /Volumes/ImmichDrive/immich/.env
# Edit .env and set a strong DB_PASSWORD
```

Or if you're reading this from the USB drive, the files are already alongside this guide.

## Step 4: Export database from VPS

```bash
ssh root@204.168.139.34 "sudo mkdir -p /opt/immich/backups && sudo chown immich:immich /opt/immich/backups && sudo -u immich podman exec -t immich_postgres pg_dumpall -c -U postgres > /opt/immich/backups/dump_final.sql"
```

## Step 5: Transfer data from VPS (this takes hours)

Start with the database dump (small, fast):

```bash
scp root@204.168.139.34:/opt/immich/backups/dump_final.sql /Volumes/ImmichDrive/immich/backups/
```

Then the photo library (~250GB — run in tmux/screen so it survives terminal close):

```bash
rsync -avz --progress root@204.168.139.34:/opt/immich/library/ /Volumes/ImmichDrive/immich/library/
```

> NOTE: If rsync is interrupted, just re-run the same command. It will resume where it left off.

## Step 6: Initialize Podman machine

```bash
podman machine init \
    --cpus 4 \
    --memory 4096 \
    --disk-size 20 \
    --volume /Volumes/ImmichDrive:/Volumes/ImmichDrive

podman machine start
```

## Step 7: Create network and start database

```bash
cd /Volumes/ImmichDrive/immich

podman network create immich_network

# Start only the database first
podman-compose up -d database

# Wait for it to be ready
sleep 15
```

## Step 8: Restore database

```bash
cat backups/dump_final.sql | podman exec -i immich_postgres psql -U postgres
```

> You may see some harmless errors about roles already existing. That's normal.

## Step 9: Start everything

```bash
podman-compose up -d
```

## Step 10: Verify

Open http://localhost:2283 in your browser.

- Log in with your existing credentials
- Check that your photos are there
- Try uploading a test photo
- Verify face recognition and search work

## Step 11: Install convenience scripts

```bash
mkdir -p ~/bin

# Copy the start/stop scripts (adjust source path as needed)
cp infrastructure/mac/immich/immich-start.sh ~/bin/
cp infrastructure/mac/immich/immich-stop.sh ~/bin/
chmod +x ~/bin/immich-start.sh ~/bin/immich-stop.sh

# Add ~/bin to PATH if not already there
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Now you can use:
- `immich-start.sh` — plug in drive, run this, done
- `immich-stop.sh` — run this before unplugging drive

---

## Daily Usage

### Start
1. Plug in ImmichDrive
2. Run `immich-start.sh`
3. Browse photos at http://localhost:2283

### Stop
1. Run `immich-stop.sh`
2. Eject USB drive in Finder
3. Unplug drive

---

## Decommission VPS (only after verifying everything works!)

1. Take a final VPS snapshot in Hetzner console (insurance)
2. Keep VPS running for 1 week as safety net
3. Delete the server in Hetzner console
4. Cancel backup addon

---

## Troubleshooting

### "USB drive not found"
Make sure the drive is plugged in and mounted. Check: `ls /Volumes/ImmichDrive`

### Containers won't start
```bash
cd /Volumes/ImmichDrive/immich
podman-compose logs -f
```

### Podman machine won't start
```bash
podman machine rm
# Then re-run Step 6
```

### Database restore errors
Harmless "role already exists" errors are normal. If you see real errors,
check that the database container is running: `podman ps`
