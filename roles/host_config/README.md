# nkg.proxmox.host_config

Post-install configuration of a running Proxmox host, ported from
`Regularmusic/iac` `proxmox_setup`. Baseline pieces default on; environment-
specific pieces default off — enable the ones you need.

| Concern | Variable | Default | Notes |
|---------|----------|---------|-------|
| Disable HA (single-node) | `host_config_disable_ha` | `true` | pve-ha-lrm/crm, corosync |
| Enable SMT | `host_config_manage_cpu` | `false` | Hyper-Threading, now + on boot |
| Storage content types | `host_config_manage_storage` | `false` | snippets/import for cloud images |
| LXC AppArmor stub fix | `host_config_lxc_apparmor_fix` | `false` | only with a blanket aa-enforce pass |
| /tmp exec override | `host_config_tmp_exec_override` | `false` | only with a noexec-/tmp hardening role |
| Networking | `host_config_manage_networking` | `false` | rewrites `/etc/network/interfaces` |
| ACME web UI cert | `host_config_acme_enabled` | `false` | Cloudflare DNS-01 |

See `defaults/main.yml` and `meta/argument_specs.yml` for the full variable set.

## Requirements

A running Proxmox VE host. Run with `become`. Collections: `ansible.utils`
(networking template uses `ipaddr`). Run `nkg.proxmox.repos` and
`nkg.proxmox.host_base` first.

## Environment-specific defaults

`host_config_manage_networking` assumes the Hetzner single-server dual-bridge
topology (public `vmbr0` + private `vmbr1` with NAT) and rewrites
`/etc/network/interfaces`; review `host_config_bridge_*` / `host_config_private_subnet`
before enabling. ACME uses Cloudflare DNS-01 — supply `host_config_acme_cf_token`
(and `host_config_acme_email`) via vault/SOPS.

## Example Playbook

```yaml
- hosts: proxmox
  become: true
  roles:
    - role: nkg.proxmox.host_config
      vars:
        host_config_manage_cpu: true
        host_config_acme_enabled: true
        host_config_acme_email: admin@example.com
        host_config_acme_domain: pve.example.com
        host_config_acme_cf_token: "{{ vault_cloudflare_token }}"
```

## Status

Ported from `Regularmusic/iac` `proxmox_setup`. The `apt.yml` (repo + nag)
piece of that role lives in `nkg.proxmox.repos`; `admin_user.yml` is not yet
ported (belongs in a future `nkg.proxmox.api_access` / user role).
