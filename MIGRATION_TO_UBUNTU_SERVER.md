# Immich Migration: Hetzner VPS → Local Ubuntu Server

Migrates Immich from the Hetzner VPS (`135.181.101.83`) to a local Ubuntu
server on the LAN (`192.168.88.16`).

**Result:** Local-only access at `http://192.168.88.16:2283`, saves ~€31/month.

---

## Architecture

```
┌──────────────┐     rsync/SCP     ┌──────────────────┐
│  Hetzner VPS  │ ──────────────── │  Ubuntu Server   │
│  135.181.101.83│   ~250GB data    │  192.168.88.16   │
│  (Source)     │                  │  (Destination)   │
└──────────────┘                  └──────────────────┘
        │                                │
        │ SSH (YubiKey)                  │ Podman or K3s
        │                                │
  ┌─────┴──────┐                  ┌──────┴──────┐
  │  Workstation│                  │  Immich      │
  │  atenga     │                  │  localhost:2283│
  │  192.168.88.27│                  │              │
  └────────────┘                  └─────────────┘
```

## Prerequisites

- SSH access to Hetzner VPS (`root@135.181.101.83`)
- SSH access to local Ubuntu server (`andreas@192.168.88.16`)
- ~250GB+ free disk space on destination
- Podman (or K3s) installed on destination

## Phase 1: Export Data from VPS

```bash
ssh root@135.181.101.83
```

Dump the Immich database:

```bash
sudo -iu immich podman exec -t immich_postgres \
  pg_dumpall -c -U postgres > /opt/immich/backups/dump_final.sql

# Verify
ls -lh /opt/immich/backups/dump_final.sql
head -20 /opt/immich/backups/dump_final.sql
exit
```

## Phase 2: Transfer Data to Ubuntu Server

Transfer the database dump:

```bash
scp root@135.181.101.83:/opt/immich/backups/dump_final.sql \
  andreas@192.168.88.16:~/immich-migration/
```

Transfer the photo library (~250GB — run in tmux, resumable):

```bash
rsync -avz --progress \
  root@135.181.101.83:/opt/immich/library/ \
  andreas@192.168.88.16:~/immich/library/
```

## Phase 3: Deploy Immich on Ubuntu Server

### Option A: Podman (manual)

On the destination server (`192.168.88.16`):

```bash
# Install Podman
sudo apt install podman podman-compose

# Create directory structure
mkdir -p ~/immich/{library,postgres,backups}

# Copy compose file and env
# (from the Infrastructure repo's infrastructure/local/immich/ or mac/immich/)
```

### Option B: K3s (Ansible-managed)

From the workstation (`atenga`, `192.168.88.27`):

```bash
cd ~/Projekt/Infrastructure
ansible-playbook -i infrastructure/local/ansible/inventory.ini \
  infrastructure/local/ansible/playbooks/k8s_server.yml \
  --limit lan_servers

ansible-playbook -i infrastructure/local/ansible/inventory.ini \
  infrastructure/local/ansible/playbooks/k8s_immich.yml \
  --limit lan_servers
```

## Phase 4: Restore Database

```bash
# Start database container first
cd ~/immich
podman-compose up -d database
sleep 15

# Restore
cat backups/dump_final.sql | podman exec -i immich_postgres psql -U postgres

# Start all services
podman-compose up -d
```

## Phase 5: Verify

```bash
# All containers running?
podman ps

# Server responding?
curl -sf http://192.168.88.16:2283/api/server/ping
# Expected: {"res":"pong"}
```

1. Open `http://192.168.88.16:2283`
2. Log in with existing credentials
3. Spot-check 5-10 albums for correct photo count
4. Verify face recognition and search
5. Upload a test photo

## Phase 6: Decommission VPS (after 1 week)

1. Take a final VPS snapshot in Hetzner console (insurance)
2. Keep VPS running for 1 week as safety net
3. Delete the Hetzner server via console
4. Cancel backup addon

## Troubleshooting

**Container won't start / volume errors:**
```bash
podman-compose logs -f
```

**Database auth errors after restore:**
```bash
podman exec immich_postgres psql -U postgres \
  -c "ALTER USER postgres PASSWORD '<password-from-env>';"
```

**rsync interrupted:**
Just re-run the same command — it resumes where it left off.
