# nkg.proxmox.host_base

Baseline preparation common to every Proxmox node: hostname, timezone, APT
cache/upgrade, base packages, and optional sysctl / ulimit tuning.

## Requirements

A Debian-based Proxmox host. Run with `become`.

## Role Variables

See `defaults/main.yml` and `meta/argument_specs.yml`.

| Variable | Default | Description |
|----------|---------|-------------|
| `host_base_hostname` | inventory short name | Hostname to set. |
| `host_base_timezone` | `Etc/UTC` | System timezone. |
| `host_base_system_upgrade` | `true` | Run a dist-upgrade. |
| `host_base_packages` | see defaults | Base packages. |
| `host_base_sysctl` | `{}` | sysctl key/value pairs. |
| `host_base_limits` | `[]` | limits.d entries. |

## Example Playbook

```yaml
- hosts: proxmox
  become: true
  roles:
    - role: nkg.proxmox.host_base
      vars:
        host_base_timezone: Europe/London
```

## Status

Scaffold. Adapted from the `common` role in homeservers/proxmox-ansible.
