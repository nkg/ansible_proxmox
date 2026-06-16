# nkg.proxmox.repos

Manage Proxmox VE APT repositories on a Debian-based Proxmox host: switch
between the enterprise and free no-subscription repos, optionally enable Ceph,
and remove the web UI "No valid subscription" nag.

## Requirements

A running Proxmox VE host (Debian). Run as root / with `become`.

## Role Variables

See `defaults/main.yml` and `meta/argument_specs.yml`. Key options:

| Variable | Default | Description |
|----------|---------|-------------|
| `repos_use_enterprise` | `false` | Enable the paid enterprise repo. |
| `repos_no_subscription` | `true` | Enable the free no-subscription repo. |
| `repos_ceph_enabled` | `false` | Enable the Ceph no-subscription repo. |
| `repos_remove_subscription_nag` | `true` | Patch out the web UI nag dialog. |

## Example Playbook

```yaml
- hosts: proxmox
  become: true
  roles:
    - role: nkg.proxmox.repos
```

## Status

Scaffold. Concrete tasks to be ported from `Regularmusic/iac` once its
outstanding PRs are merged.
