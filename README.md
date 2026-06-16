# Ansible Collection — nkg.proxmox

[![Lint](https://github.com/nkg/ansible_proxmox/actions/workflows/lint.yml/badge.svg)](https://github.com/nkg/ansible_proxmox/actions/workflows/lint.yml)
[![Molecule](https://github.com/nkg/ansible_proxmox/actions/workflows/molecule.yml/badge.svg)](https://github.com/nkg/ansible_proxmox/actions/workflows/molecule.yml)

Ansible roles to provision, configure, secure and harden [Proxmox VE](https://www.proxmox.com/)
servers — from a bare-metal Hetzner install through hardening and intrusion
detection.

## Roles

| Role | Purpose |
|------|---------|
| [`repos`](roles/repos) | Manage Proxmox APT repos (enterprise ↔ no-subscription, Ceph) and remove the web-UI subscription nag. |
| [`host_base`](roles/host_base) | Baseline host prep: hostname, timezone, packages, sysctl, limits. |
| [`host_config`](roles/host_config) | Post-install config: single-node HA, networking, LXC/AppArmor, `/tmp` exec, ACME cert. |
| [`security`](roles/security) | Apply [`konstruktoid.hardening`](https://github.com/konstruktoid/ansible-role-hardening) with Proxmox-safe overrides. |
| [`crowdsec`](roles/crowdsec) | Install/configure [CrowdSec](https://www.crowdsec.net/) engine + firewall bouncer. |
| [`api_access`](roles/api_access) | Create an IaC admin user + Proxmox API token for Terraform/OpenTofu. |
| [`hetzner_image`](roles/hetzner_image) | Provision Proxmox onto a Hetzner dedicated server (rescue mode) via QEMU autoinstall. **Destructive.** |

Each role has its own README with the full variable reference.

## Requirements

- Ansible / `ansible-core` >= 2.15 (the `security` role needs >= 2.18 via konstruktoid).
- Target hosts run Proxmox VE on Debian (Bookworm/Trixie).
- Collections: `ansible.posix`, `community.general`, `ansible.utils` (installed
  automatically with the collection). The `security` role additionally needs the
  standalone `konstruktoid.hardening` role:

  ```bash
  ansible-galaxy collection install -r requirements.yml
  ansible-galaxy role install -r requirements.yml
  ```

## Install

```bash
ansible-galaxy collection install git+https://github.com/nkg/ansible_proxmox.git
```

## Quick start

Configure and harden an already-running Proxmox host:

```yaml
- hosts: proxmox
  become: true
  roles:
    - nkg.proxmox.repos
    - nkg.proxmox.host_base
    - role: nkg.proxmox.host_config
      vars:
        host_config_lxc_apparmor_fix: true   # if you also run the security role
        host_config_tmp_exec_override: true
    - nkg.proxmox.security
    - nkg.proxmox.crowdsec
```

## Testing

Roles with a container-testable scenario (`repos`, `host_base`, `host_config`,
`crowdsec`) are verified with [Molecule](https://ansible.readthedocs.io/projects/molecule/)
on Docker in CI:

```bash
pip install -r requirements.txt
cd roles/host_base && molecule test
```

`api_access`, `security` and `hetzner_image` require a real Proxmox host / QEMU
and are tested out of band.

## License

GPL-2.0-or-later. See [LICENSE](LICENSE).
