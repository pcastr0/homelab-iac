# Homelab Infrastructure-as-Code

Infrastructure automation for a small homelab network using **Ansible** (active) with planned **OpenTofu** support. Manages network infrastructure across two VLANs: `10.0.10.0/24` (homelab) and `10.0.99.0/24` (NAS).

## Project Status

- **Ansible**: Active — fully configured and operational
- **OpenTofu**: Planned — directory reserved, not yet implemented

## Repository Structure

```
homelab-iac/
├── README.md                           (this file)
├── CLAUDE.md                           (development instructions)
├── ansible/
│   ├── ansible.cfg                     (Ansible configuration)
│   ├── playbooks/
│   │   ├── site.yml                    (main site playbook)
│   │   └── bootstrap.yml               (control node bootstrap)
│   ├── inventories/
│   │   └── production/
│   │       └── hosts.yaml              (inventory: hosts, groups, vars)
│   └── roles/
│       └── proxmox-host/               (role for Proxmox VE hosts)
│           ├── defaults/
│           │   └── main.yml            (role defaults)
│           ├── tasks/
│           │   └── main.yml            (role tasks)
│           ├── handlers/
│           │   └── main.yml            (SSH restart handler)
│           └── templates/
│               └── sshd_config.j2      (SSH hardening template)
└── opentofu/                           (planned)
```

## Network Infrastructure

| Group | Host | IP | OS | Role | User |
|-------|------|----|----|------|------|
| `management` | `thinkpad-t480` | `10.0.10.20` | Debian/Linux | Ansible control node | trickooo |
| `proxmox_hosts` | `desktop-pc` | `10.0.10.12` | Proxmox VE 9 | Virtualization host | trickooo |
| `lxc` | `adguard` | `10.0.10.5` | Linux | DNS/Ad-blocking container | root |
| `nas` | `zimablade` | `10.0.99.10` | Linux | Network-attached storage | trickooo |

## Prerequisites

### Control Node (ThinkPad T480)

- Debian/Linux-based OS
- Ansible 2.9+ installed
- Python 3 on all target hosts
- SSH key at `~/.ssh/ansible_homelab` with passphrase management
- Network access to all hosts via `10.0.10.0/24`

### Target Hosts

- SSH service running and accessible from control node
- Python 3 installed
- User `trickooo` with sudo/become privileges (except adguard root user)
- SSH public key from control node in `~/.ssh/authorized_keys`

### Ansible Configuration

The `ansible.cfg` file automatically configures:

- **Inventory**: `inventories/production/hosts.yaml`
- **Remote user**: `trickooo` (overrideable per-host)
- **Private key**: `~/.ssh/ansible_homelab`
- **SSH args**: Connection multiplexing for performance
- **Callbacks**: YAML output formatting
- **Fact caching**: Cached in `/tmp/ansible_facts`

All commands must be run from the `ansible/` directory.

## Quick Start

### 1. Bootstrap the Control Node

Initialize the ThinkPad T480 with system updates and service verification:

```bash
cd ansible
ansible-playbook playbooks/bootstrap.yml
```

**What it does**:
- Updates system packages (`apt upgrade`)
- Verifies Docker is running and enabled
- Verifies SSH service is running and enabled
- Displays host information (hostname, OS, IP)

### 2. Deploy Full Site Playbook

Apply all roles and configurations to managed hosts:

```bash
cd ansible
ansible-playbook playbooks/site.yml
```

**What it does**:
- Applies the `proxmox-host` role to `desktop-pc`
- Configures SSH hardening (key-only authentication)
- Applies kernel hardening via sysctl
- Sets timezone and system packages
- See "Common Playbook Tags" for selective execution

### 3. Verify Hosts Are Reachable

Test SSH connectivity to all inventory hosts:

```bash
cd ansible
ansible all -m ping
```

Expected output: `pong` from each host indicates successful connectivity.

## Common Commands

All commands execute from the `ansible/` directory.

### Playbook Execution

Run the full site playbook:

```bash
cd ansible && ansible-playbook playbooks/site.yml
```

Run with dry-run (check mode) to preview changes:

```bash
cd ansible && ansible-playbook playbooks/site.yml --check
```

Run only specific tags:

```bash
cd ansible && ansible-playbook playbooks/site.yml --tags "ssh,security"
```

Target a single host:

```bash
cd ansible && ansible-playbook playbooks/site.yml --limit desktop-pc
```

Bootstrap the control node:

```bash
cd ansible && ansible-playbook playbooks/bootstrap.yml
```

### Inventory and Connectivity

List all managed hosts:

```bash
cd ansible && ansible all --list-hosts
```

Ping all hosts to verify connectivity:

```bash
cd ansible && ansible all -m ping
```

Gather facts from all hosts:

```bash
cd ansible && ansible all -m setup
```

Gather facts from a specific host:

```bash
cd ansible && ansible desktop-pc -m setup
```

### Debugging and Validation

Lint playbooks and roles for syntax errors:

```bash
ansible-lint playbooks/site.yml
```

Run a single task with increased verbosity:

```bash
cd ansible && ansible-playbook playbooks/site.yml -vvv
```

Check host variables for a specific host:

```bash
cd ansible && ansible desktop-pc -m debug -a 'var=hostvars[inventory_hostname]'
```

### Ad-hoc Tasks

Execute a command on all hosts:

```bash
cd ansible && ansible all -m command -a 'uname -a'
```

Copy a file to all hosts:

```bash
cd ansible && ansible all -m copy -a 'src=/local/path dest=/remote/path'
```

Install a package on a specific host:

```bash
cd ansible && ansible desktop-pc -m apt -a 'name=vim state=present' -b
```

## Playbooks

### playbooks/bootstrap.yml

**Target**: `thinkpad-t480` (control node)
**Privilege**: Sudo (become: yes)

Initializes the Ansible control node with:
- System package updates (`apt upgrade`)
- Docker service verification and enablement
- SSH service verification and enablement
- Host information display (hostname, OS, IP)

**Usage**:
```bash
cd ansible && ansible-playbook playbooks/bootstrap.yml
```

### playbooks/site.yml

**Target**: `proxmox_hosts` group (desktop-pc)
**Privilege**: Sudo (become: true)
**Role**: `proxmox-host`
**Tags**: `proxmox`

Main infrastructure playbook applying the proxmox-host role to all Proxmox VE hosts.

**Usage**:
```bash
cd ansible && ansible-playbook playbooks/site.yml [--tags "tag1,tag2"] [--check]
```

## Roles

### proxmox-host

Configures and hardens **Proxmox VE** hosts for homelab virtualization.

**Applied to**: `proxmox_hosts` group (desktop-pc)

#### Tasks Included

| Task | Tags | Status | Purpose |
|------|------|--------|---------|
| Disable Proxmox enterprise repo | `repos` | Active | Disables Proxmox paid repo, prepares for community repo |
| Upgrade all packages | `packages`, `updates` | Active | Applies system security patches and updates |
| Set system timezone | `system` | Active | Configures timezone to `{{ proxmox_timezone }}` (default: Asia/Manila) |
| Harden SSH | `ssh`, `security` | Active | Applies key-only authentication, no root password, restricted access |
| UFW firewall | `firewall` | Commented | Default-deny incoming, allow SSH and Proxmox Web UI from VLAN 10 |
| fail2ban | `security` | Commented | Rate-limiting SSH brute-force attempts |
| Kernel hardening | `sysctl`, `security` | Active | Enables TCP SYN cookies, reverse path filtering, restricts kernel pointers |
| Essential packages | `packages` | Active | Installs curl, wget, git, vim, htop, net-tools |

#### SSH Hardening Configuration

The role deploys `sshd_config.j2` with security-first settings:

- **PermitRootLogin**: no (root kept in AllowUsers for Proxmox API compatibility)
- **PasswordAuthentication**: no (key-only)
- **PubkeyAuthentication**: yes
- **X11Forwarding**: no
- **MaxAuthTries**: 3
- **MaxSessions**: 5
- **ClientAliveInterval**: 300s
- **LoginGraceTime**: 30s
- **AllowUsers**: `trickooo root`
- **Banner**: `/etc/issue.net`

Validation: `sshd -t` tests config before applying

Handler: SSH service restarts only if config changes

#### Kernel Hardening

Applied via `ansible.posix.sysctl`:

```
net.ipv4.tcp_syncookies = 1        # SYN flood protection
net.ipv4.conf.all.rp_filter = 1    # Reverse path filtering
kernel.dmesg_restrict = 1          # Restrict dmesg to root
kernel.kptr_restrict = 2           # Hide kernel pointers from non-root
```

#### Variables (Defaults)

Located in `roles/proxmox-host/defaults/main.yml`. Override per-host in inventory:

```yaml
proxmox_node_name: goliath                    # Node hostname
proxmox_mgmt_ip: 10.0.10.11                   # Management IP
proxmox_storage_vm: local-lvm                 # VM storage pool
proxmox_storage_os: local                     # OS storage pool
proxmox_backup_dest: 10.0.99.10               # NAS IP for backups
proxmox_timezone: Asia/Manila                 # System timezone
proxmox_ntp_servers:                          # NTP pool servers
  - 0.asia.pool.ntp.org
  - 1.asia.pool.ntp.org
proxmox_ssh_allowed_src: '10.0.10.0/24'       # SSH allowed CIDR
```

**Override example** in `hosts.yaml`:
```yaml
desktop-pc:
  ansible_host: 10.0.10.12
  ansible_user: trickooo
  node_role: proxmox_host
  proxmox_timezone: UTC                       # Override default
```

#### Commented-out Features

The following are disabled by default and can be uncommented to enable:

- **Proxmox no-subscription repo**: Community repo alternative (commented in tasks)
- **UFW firewall**: Uncomment for stateful firewall with VLAN 10 restrictions
- **fail2ban**: Uncomment to enable SSH rate-limiting

To enable, uncomment lines in `roles/proxmox-host/tasks/main.yml` and run with `--tags firewall` or `--tags security`.

## Inventory Overview

Location: `ansible/inventories/production/hosts.yaml`

### Groups and Hosts

#### management
Ansible control node and management infrastructure.

| Host | IP | User | Role |
|------|----|----|------|
| thinkpad-t480 | 10.0.10.20 | trickooo | Ansible control node |

#### proxmox_hosts
Proxmox VE virtualization hosts.

| Host | IP | User | Role |
|------|----|----|------|
| desktop-pc | 10.0.10.12 | trickooo | Proxmox VE 9 hypervisor |

#### lxc
LXC/container hosts.

| Host | IP | User | Role |
|------|----|----|------|
| adguard | 10.0.10.5 | root | DNS/ad-blocking LXC container |

#### nas
Network-attached storage.

| Host | IP | User | Role |
|------|----|----|------|
| zimablade | 10.0.99.10 | trickooo (with sudo) | NAS VLAN storage |

### Global Variables

Applied to all hosts via `all.vars`:

```yaml
ansible_user: trickooo                    # Default remote user
ansible_ssh_private_key_file: ~/.ssh/ansible_homelab
ansible_python_interpreter: /usr/bin/python3
```

**Per-host overrides**: Set `ansible_user` or `ansible_ssh_private_key_file` in host definition.

## Tags Reference

Tags enable selective playbook execution. Run with `--tags "tag1,tag2"`.

| Tag | Affects | Purpose |
|-----|---------|---------|
| `repos` | APT repository configuration | Disable enterprise repo, prepare for community sources |
| `packages` | Package installation | Install essential utilities and tools |
| `updates` | Package upgrades | Apply security patches and OS updates |
| `system` | System configuration | Timezone, hostname, locale settings |
| `ssh` | SSH configuration | Deploy hardened sshd_config template |
| `security` | All hardening tasks | SSH, kernel hardening, fail2ban, firewall |
| `sysctl` | Kernel parameters | Network and security sysctl tuning |
| `firewall` | UFW rules | Stateful firewall and access control |
| `proxmox` | Entire site.yml | Apply full proxmox-host role |

**Examples**:

Apply only security updates:
```bash
cd ansible && ansible-playbook playbooks/site.yml --tags "updates"
```

Update kernel hardening and firewall:
```bash
cd ansible && ansible-playbook playbooks/site.yml --tags "sysctl,firewall"
```

Skip firewall tasks:
```bash
cd ansible && ansible-playbook playbooks/site.yml --skip-tags "firewall"
```

## Future: OpenTofu Support

The `opentofu/` directory is reserved for infrastructure-as-code using OpenTofu (Terraform alternative). Currently empty.

**Planned features**:
- Infrastructure provisioning (VMs, LXC containers, storage)
- Network configuration (VLANs, routing, firewall rules)
- DNS and DHCP management
- State management and drift detection

## Development Guidelines

See [CLAUDE.md](./CLAUDE.md) for development instructions, including:
- Playbook development workflow
- Role testing and validation
- Ansible linting and best practices
- Debugging commands and troubleshooting

## Requirements

- **Ansible**: 2.9+
- **Python**: 3.7+ on control node and all target hosts
- **SSH**: Key-based authentication configured
- **Network**: Access to `10.0.10.0/24` and `10.0.99.0/24`

## Troubleshooting

### SSH Connection Issues

Verify the SSH key is present and has correct permissions:
```bash
ls -la ~/.ssh/ansible_homelab
chmod 600 ~/.ssh/ansible_homelab
```

Test SSH connectivity directly:
```bash
ssh -i ~/.ssh/ansible_homelab trickooo@10.0.10.12
```

### Fact Caching Issues

Clear cached facts if encountering stale data:
```bash
rm -rf /tmp/ansible_facts
```

Re-run playbook to regenerate facts.

### Syntax Validation

Lint playbooks before running:
```bash
ansible-lint playbooks/site.yml
```

Check playbook syntax:
```bash
cd ansible && ansible-playbook playbooks/site.yml --syntax-check
```

### Host Unreachable

Verify host is up and network accessible:
```bash
cd ansible && ansible desktop-pc -m ping -vvv
```

Check inventory for correct IP and user:
```bash
cd ansible && ansible-inventory --host desktop-pc
```

## Contributing

When modifying playbooks or roles:

1. Run in check mode first: `--check`
2. Lint with `ansible-lint`
3. Test against a host: `--limit hostname`
4. Document changes in relevant role README or CLAUDE.md
5. Update this README if adding new roles or playbooks

## License

See repository root for license information.

## Contact

Ansible control node: thinkpad-t480 (10.0.10.20)
Maintainer: trickooo
