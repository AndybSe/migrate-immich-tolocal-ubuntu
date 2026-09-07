# Infrastructure Overview — Hetzner VPS → Local Ubuntu Server

This document maps the source (Hetzner VPS) and destination (local Ubuntu server) for the Immich migration.

## Source: Hetzner VPS

| Property | Value |
|---|---|
| Provider | Hetzner Cloud |
| Server name | `ubuntu-4gb-hel1-1` |
| IP | `135.181.101.83` |
| Type | `cpx22` (2 vCPUs, 4GB RAM) |
| OS | Ubuntu 24.04 (noble) |
| Location | Helsinki (`hel1`) |
| Role | Original Immich host (being decommissioned) |
| Terraform state | Local (`infrastructure/hetzner/terraform/`) |
| Ansible inventory | Dynamic, written by Terraform after apply |

### VPS Provisioning (Terraform)

- **`infrastructure/hetzner/terraform/main.tf`** — Defines the `hcloud_server` resource (Ubuntu 24.04, `cpx22`, keep_disk on, ignores image/name changes to prevent rebuild)
- **`infrastructure/hetzner/terraform/variables.tf`** — Required vars: `hcloud_token`, `ssh_public_key`. Optional: `admin_ip` (default `46.59.10.156/32`), `location` (default `hel1`)
- **`infrastructure/hetzner/terraform/firewall.tf`** — UFW/firewall rules
- **`infrastructure/hetzner/terraform/inventory.tf`** — Generates Ansible inventory after apply
- Makefile targets: `make vps-init`, `make vps-plan`, `make vps-apply`, `make vps-destroy`

### VPS Configuration (Ansible)

- **`infrastructure/hetzner/ansible/playbooks/setup.yml`** — Main VPS playbook
- **`infrastructure/shared/ansible/roles/`** — Shared role library used by VPS, workstation, and remote hosts
  - `immich_podman/` — Ansible role that deployed Immich via Podman on the VPS
  - `rclone/` — Rclone for Hetzner Storage Box and Google Drive mounts

## Destination: Local Ubuntu Server

| Property | Value |
|---|---|
| Hostname | *(configured in Ansible inventory as `lan_servers`)* |
| IP | `192.168.88.16` |
| User | `andreas` |
| Role | New Immich host (receiving migrated data) |
| Ansible group | `[lan_servers]` in `infrastructure/local/ansible/inventory.ini` |

### Destination Configuration (Ansible)

- **`infrastructure/local/ansible/playbooks/k8s_immich.yml`** — Deploys Immich to the LAN server via K3s using the `immich_k3s` role
- **`infrastructure/local/ansible/playbooks/k8s_server.yml`** — Sets up K3s on the LAN server
- **`infrastructure/shared/ansible/roles/immich_k3s/`** — Ansible role for deploying Immich on K3s (the destination deployment method)
- **`infrastructure/local/ansible/inventory.ini`** — Defines `[lan_servers]` group with `192.168.88.16`

## Workstation (Orchestration Machine)

| Property | Value |
|---|---|
| Hostname | `atenga` |
| IP | `192.168.88.27` |
| OS | Linux Mint 22.3 "Zena" (Cinnamon), based on Ubuntu 24.04 Noble |
| Role | Controls deployments via Ansible, stores Infrastructure repo |

## Immich Migration: VPS → Local Ubuntu Server

Migration from Hetzner VPS (€31/mo) to the local Ubuntu server at `192.168.88.16` to save costs and run locally.

### Migration guides

| Guide | Location | Description |
|---|---|---|
| VPS → MacBook | `MIGRATION_VPS_TO_MACBOOK.md` | Original guide for MacBook destination (reference only) |
| Mac-specific | `MIGRATION_GUIDE_MAC.md` | MacBook-specific version (reference only) |

### Migration steps (summary)

1. Install Podman (or K3s) on the destination Ubuntu server (`192.168.88.16`)
2. Export PostgreSQL dump from VPS: `pg_dumpall -c -U postgres`
3. Transfer dump + ~250GB photo library via rsync (resumable)
4. Deploy Immich using Ansible (`make` targets) or manually via podman-compose
5. Restore database from SQL dump
6. Start all services and verify at `http://192.168.88.16:2283`

## Monitoring & Operations

- **`reports/server_report_ubuntu-4gb-hel1-1.md`** — VPS system report: OS, kernel, running services, UFW status, Fail2Ban stats
- **`reports/gateway_report.md`** — Local gateway (`192.168.88.1`) connectivity

## Backup Strategy (Restic)

Backups use Restic via SSH to Hetzner Storage Box, with YubiKey-backed SSH agent:

- **`automation/hetzner/`** — Shell scripts wrapping all restic operations
- Makefile targets: `make backup`, `make check-backup`, `make backup-verify-deep`, `make backup-stats`, `make prune-backup`
- GFS (Grandfather-Father-Son) retention policy

## Tools & Scripts

- **`scripts/SafeImport/`** — C# tool for safe photo import with checksum verification
- **`scripts/verify_samples.py`** — Verify sample photos in library
- **`scripts/reclaim_ssd_space.py`** — SSD space reclamation
- **`scripts/check_apt_codename.py`** — Pre-commit hook for correct APT codenames in templates
- **`scripts/pulse_watch.py`** — System monitoring
- **`scripts/net_watch.sh`** — Network monitoring

## Test Suite

- **`tests/`** — Pytest suite covering: security compliance, agent safety, mail DNS, iOS generator, Rust tooling, pre-commit hooks
- Run with: `make test` (Ansible + Terraform) or `make test-all` (+ pytest + security hooks)
- Individual tests: `PYTHONPATH=. python3 -m pytest tests/test_<name>.py::test_<fn>`

## Secrets Management

- **Ansible Vault** — `.vault_pass` (gitignored), used via `--ask-vault-pass`
- **SOPS + age (YubiKey)** — `.sops.yaml` for `*.env`, `*.yaml`, `*.json` encryption
- **Gitleaks** — Pre-commit hook prevents secrets from being committed

## Tech Stack

- **Terraform** v1.14.3 (vendored binary in repo)
- **Ansible** — 3 sites sharing one role library
- **Podman / podman-compose** — Container runtime for Immich (VPS original)
- **K3s** — Kubernetes distribution for Immich on destination server
- **PostgreSQL** with pgvecto-rs extension (vector search for ML)
- **Redis** — Caching for Immich
- **Restic** — Encrypted backups to Hetzner Storage Box
- **Rclone** — Cloud storage mounts (Google Drive, Hetzner Storage Box)
