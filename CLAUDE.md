# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

Homelab infrastructure-as-code using **Ansible** (active) and **OpenTofu** (planned, currently empty). Targets a small homelab network at `10.0.10.0/24` and a NAS VLAN at `10.0.99.0/24`.

## Ansible

All commands run from `ansible/` directory. The `ansible.cfg` sets the inventory, remote user (`trickooo`), and SSH key (`~/.ssh/ansible_homelab`) automatically.

```bash
# Run full site playbook
cd ansible && ansible-playbook playbooks/site.yml

# Bootstrap the control node (ThinkPad T480)
cd ansible && ansible-playbook playbooks/bootstrap.yml

# Dry-run (check mode)
cd ansible && ansible-playbook playbooks/site.yml --check

# Run specific tags only
cd ansible && ansible-playbook playbooks/site.yml --tags "ssh,security"

# Target a single host
cd ansible && ansible-playbook playbooks/site.yml --limit desktop-pc

# Ad-hoc ping all hosts
cd ansible && ansible all -m ping

# Lint playbooks/roles
ansible-lint playbooks/site.yml
```

## Inventory Structure

| Group | Host | IP | Notes |
|---|---|---|---|
| `management` | `thinkpad-t480` | `10.0.10.20` | Ansible control node |
| `proxmox_hosts` | `desktop-pc` | `10.0.10.12` | Proxmox VE 9 |
| `lxc` | `adguard` | `10.0.10.5` | Runs as root |
| `nas` | `zimablade` | `10.0.99.10` | Separate NAS VLAN |

## Role: `proxmox-host`

Applied via `site.yml` to the `proxmox_hosts` group. Covers:
- Disabling the Proxmox enterprise repo (no-subscription repo block is commented out — intentional)
- SSH hardening via `sshd_config.j2` template (key-only, no root password, `AllowUsers trickooo root`)
- Kernel hardening via `sysctl` (syncookies, rp_filter, dmesg/kptr restrict)
- Timezone set to `Asia/Manila` by default via `proxmox_timezone` var
- UFW and fail2ban tasks exist but are commented out — not yet enabled

Role defaults live in `roles/proxmox-host/defaults/main.yml`. Override per-host in the inventory.

## Tags Reference

| Tag | What it touches |
|---|---|
| `repos` | APT source lists |
| `packages`, `updates` | APT package installs/upgrades |
| `system` | Timezone |
| `ssh`, `security` | SSH hardening |
| `sysctl` | Kernel parameters |
| `firewall` | UFW rules (currently commented out) |
