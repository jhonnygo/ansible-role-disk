# Ansible Role: **disk**
[![Style: ansible-lint](https://img.shields.io/badge/style-ansible--lint-green)](#lint)
[![Tests: molecule](https://img.shields.io/badge/tests-molecule-blue)](#testing)
[![License: MIT](https://img.shields.io/badge/license-MIT-informational)](LICENSE)

Ansible role to **safely and idempotently prepare and mount Linux disks**:
- Detects partition label (`gpt` or `dos/msdos`) and creates it if needed.
- Creates partitions with `community.general.parted` using ranges (`0%`, `25%`, `GiB`, etc.).
- Formats partitions with the specified FS (default `ext4`).  
- Mounts and **persists** in `/etc/fstab` with `ansible.posix.mount`.
- Includes **optional data migration** from an existing path (e.g., `/var/log`) with `rsync`.
- Hooks to **stop/start services** during migrations (journald, rsyslog, …).
- Supports `sdX` and `nvmeXnY` disks (automatic handling of the `p` suffix for NVMe/mmcblk).

---

## Table of Contents
- [Compatibility & Requirements](#compatibility--requirements)
- [Installation](#installation)
- [Variables](#variables)
  - [Structure of `disk_devices`](#structure-of-disk_devices)
  - [Service hints by path](#service-hints-by-path)
  - [Control variables & defaults](#control-variables--defaults)
- [Usage Examples](#usage-examples)
  - [DB Layout (4 partitions)](#db-layout-4-partitions)
  - [Web Layout (2 partitions + /var/log migration)](#web-layout-2-partitions--varlog-migration)
- [Tags](#tags)
- [Idempotence & Safety](#idempotence--safety)
- [Troubleshooting](#troubleshooting)
- [Testing](#testing)
- [License & Author](#license--author)

---

## Compatibility & Requirements

**Systems**: Debian/Ubuntu (and derivatives).  
**Ansible Collections**:  
- `ansible.posix`
- `community.general`

**Target packages** (installed automatically if OS family is supported):  
`parted`, `e2fsprogs`, `rsync` (the latter only if there are migrations).

Install the collections with:

```yaml
# requirements.yml
collections:
  - name: ansible.posix
  - name: community.general
```

```bash
ansible-galaxy collection install -r requirements.yml
```

---

## Installation

### A) Using Git directly (roles/requirements.yml)
```yaml
roles:
  - name: disk
    src: https://github.com/jhonnygo/ansible-role-disk.git
    version: latest
```
```bash
ansible-galaxy role install -r roles/requirements.yml
```

### B) Ansible Galaxy (if you publish it)
```bash
ansible-galaxy role install jhonnygo.disk
```

---

## Variables

### Structure of `disk_devices`

Define each disk and its partitions. The role will iterate over this list and apply the layout idempotently.

```yaml
disk_devices:
  - device: /dev/nvme1n1        # Disk to prepare
    label: gpt                  # (optional) gpt | dos (msdos). Default: disk_label_default
    parts:
      - number: 1               # Partition number
        start: "0%"             # Start (percentage or unit: "1MiB", "10GiB", …)
        end:   "25%"            # End
        fs: ext4                # (optional) Filesystem. Default: disk_fs_default
        mkfs_opts: ""           # (optional) mkfs flags (e.g., "-F")
        mount:
          path: /var/lib/mysql  # Mount point
          opts: defaults        # fstab options
          create_mode: "0755"   # If the dir is created
          create_owner: root
          create_group: root
          migrate_from: ""      # Path to migrate if present (e.g., "/var/log")
```

> **Notes**  
> - The role automatically computes the **partition suffix** (`p`) for `nvme/mmcblk` and omits it for `sdX` (e.g., `/dev/nvme1n1p1` vs `/dev/sda1`).  
> - After partitioning, the role runs `partprobe` and `udevadm settle` to force the kernel/udev to see the new partitions before continuing.

### Service hints by path

During a migration (when `migrate_from` is set), the role can stop/start services associated with that path:

```yaml
disk_service_hints:
  "/var/log":
    stop:  [rsyslog, systemd-journald]
    start: [systemd-journald, rsyslog]
```

You can extend/override this structure in `group_vars/host_vars` for other paths (for example, database paths).

### Control variables & defaults

```yaml
# defaults/main.yml
disk_devices: []               # List of disks to prepare
disk_label_default: gpt        # Default label if d.label is not defined
disk_fs_default: ext4          # Default FS if p.fs is not defined
disk_remove_lostfound: true    # Remove lost+found after mount (cosmetic)
disk_debug: false              # Enable debug messages (lsblk PTTYPE, etc.)
```

---

## Usage Examples

### DB Layout (4 partitions)

```yaml
- hosts: role_db
  become: true
  roles:
    - role: disk
      vars:
        disk_devices:
          - device: /dev/nvme1n1
            label: gpt
            parts:
              - { number: 1, start: "0%",  end: "25%", fs: ext4, mount: { path: /var/lib/mysql,       opts: defaults } }
              - { number: 2, start: "25%", end: "50%", fs: ext4, mount: { path: /var/lib/postgresql,  opts: defaults } }
              - { number: 3, start: "50%", end: "75%", fs: ext4, mount: { path: /var/lib/mongodb,     opts: defaults } }
              - { number: 4, start: "75%", end: "100%",fs: ext4, mount: { path: /var/log,             opts: defaults, migrate_from: "/var/log" } }
```

### Web Layout (2 partitions + `/var/log` migration)

```yaml
- hosts: role_app
  become: true
  roles:
    - role: disk
      vars:
        disk_devices:
          - device: /dev/nvme1n1
            parts:
              - { number: 1, start: "0%",  end: "50%",  fs: ext4, mount: { path: /var/www, opts: defaults } }
              - { number: 2, start: "50%", end: "100%", fs: ext4, mount: { path: /var/log, opts: defaults, migrate_from: "/var/log" } }
```

---

## Tags

- `disk` — general role tag  
- `disk:probe` — detections and kernel notifications (`lsblk`, `partprobe`, `udevadm`)  
- `disk:mklabel` — disk label creation  
- `disk:parted` — partition creation  
- `disk:wait` — waits for partition nodes to appear  
- `disk:partition` — formatting, mounting, fstab, and migrations  
- `debug` — extra output if `disk_debug: true`  

Run only partitioning and mounting, for example:
```bash
ansible-playbook site.yml -l role_db -t "disk:parted,disk:partition"
```

---

## Idempotence & Safety

- `mkfs` is **not** forced if the FS already exists on the partition (the `filesystem` module is idempotent).  
- Mounting uses `state: mounted` and is persisted in `/etc/fstab`.  
- If `migrate_from` is set, it uses `rsync -aXS --numeric-ids` against a temporary mount and cleans up resources.  
- `partprobe` and `udevadm settle` are used to synchronize with kernel/udev after partitioning changes.  
- Services defined in `disk_service_hints` are stopped before migration and started at the end.

---

## Troubleshooting

**`PTTYPE=dos` (msdos) is detected but I want GPT**  
Set `label: gpt` on the device or adjust `disk_label_default: gpt`. The role will create the label if it differs from the detected one:

```yaml
disk_devices:
  - device: /dev/nvme1n1
    label: gpt
    parts: ...
```

**Partitions take a while to appear**  
The role already runs `partprobe` and `udevadm settle`, and waits for each `/dev/...` node to exist. If your platform still needs more time, increase the global `timeout` via `ansible.cfg` (SSH timeouts) or add an extra wait task.

**NVMe vs sdX**  
The role automatically computes the `p` suffix for NVMe/mmcblk (`/dev/nvme1n1p1`) and not for `sdX` (`/dev/sda1`).

**`lost+found`**  
It is removed if `disk_remove_lostfound: true` (purely cosmetic, not functional).

**Check mode**  
Destructive tasks are guarded with `not ansible_check_mode` when appropriate; under `--check` you’ll see the intent but the disk will not be altered.

---

## Testing

Includes a **Molecule**-compatible skeleton (add your Docker/Podman/EC2 scenario).

```bash
pip install molecule molecule-plugins[docker] ansible-lint
molecule test
```

---

## License & Author

**License:** MIT  
**Author:** Your Name — <suport@jhoncytech.com>  
**GitHub:** https://github.com/jhonnygo/ansible-rol-disk.git


## Test Matrix / Verified Environments

> This role is validated incrementally. Below are the tested environments with their status.  
> If an environment is missing, it means “pending test.”

| Environment / Platform                      | OS / AMI / Image                  | Ansible | Result | Notes |
|---------------------------------------------|-----------------------------------|---------|--------|-------|
| **Local – Vagrant (VirtualBox)**            | Ubuntu 22.04 LTS                  | 2.15+   | ✅ OK  | Simulated `nvme` and `sda` partitioning tests. |
| **Local – Docker**                           | ubuntu:22.04 (privileged)         | 2.15+   | ✅ OK  | Requires `--privileged` to allow `parted/udev`. |
| **AWS EC2**                                  | Ubuntu 22.04 LTS (t3/t3a)         | 2.15+   | ✅ OK  | `nvme1n1` disks; see udev/partprobe notes. |
| **AWS EC2**                                  | Ubuntu 20.04 LTS                  | 2.15+   | 🟡 Pending |                                    |
| **AWS EC2**                                  | Amazon Linux 2 / 2023             | 2.15+   | 🟡 Pending | Adjust package manager (yum/dnf). |
| **GCP / Azure**                              | Ubuntu 22.04 LTS                  | 2.15+   | 🟡 Pending |                                    |
| **Bare-metal / VMware / Proxmox**            | Debian 12                         | 2.15+   | 🟡 Pending |                                    |
| **RHEL / Rocky / Alma**                      | 8.x / 9.x                         | 2.15+   | 🟡 Pending | Switch package manager to `yum/dnf`. |

### How to reproduce local tests

**Vagrant (VirtualBox)**
```bash
vagrant init ubuntu/jammy64
# In the Vagrantfile, add an extra disk or use a box that exposes /dev/sdb
vagrant up
ansible-playbook -i inventory vagrant_storage.yml -l default
```

**Docker (privileged)**
```bash
docker run --rm -it --privileged -v $(pwd):/work -w /work ubuntu:22.04 bash
# inside the container:
apt-get update && apt-get install -y ansible parted util-linux udev
ansible-playbook -i inventory docker_storage.yml
```

> **Tip:** If the environment uses NVMe (e.g., EC2), partition nodes will be `/dev/nvme1n1p1`.
> On SATA/virtio they will be `/dev/sdb1`. The role detects both cases automatically.

---
