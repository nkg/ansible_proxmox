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

Directly:

```bash
ansible-galaxy collection install git+https://github.com/nkg/ansible_proxmox.git
```

Or from a consuming repo's `requirements.yml`:

```yaml
collections:
  - name: https://github.com/nkg/ansible_proxmox.git
    type: git
    version: main   # or a tag once releases are cut
```

## Playbooks

The collection ships ready-to-run playbooks:

```bash
# configure + harden an existing Proxmox host (group: proxmox)
ansible-playbook nkg.proxmox.site -i your-inventory.yml

# DESTRUCTIVE: install Proxmox onto a Hetzner server in rescue mode
ansible-playbook nkg.proxmox.bootstrap_hetzner -i rescue-inventory.yml
```

## Quick start

`nkg.proxmox.site` is just this sequence — copy it into your own play to
customise:

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

CI runs via the org's shared reusable workflows
([`nkg/github-actions`](https://github.com/nkg/github-actions)): `ansible.yml`
(ansible-lint) and `molecule.yml` (matrix over the testable roles), both using
`uvx` so the tooling is always current.

Roles with a container-testable scenario (`repos`, `host_base`, `host_config`,
`crowdsec`) are verified with [Molecule](https://ansible.readthedocs.io/projects/molecule/)
on Docker. Locally, with [uv](https://docs.astral.sh/uv/) installed:

```bash
# lint
uvx --with ansible ansible-lint

# run one role's molecule scenario
cd roles/host_base
uvx --with ansible-core --with 'molecule-plugins[docker]' molecule test
```

`api_access`, `security` and `hetzner_image` require a real Proxmox host / QEMU
and are tested out of band.

To validate the config roles against a real node before trusting them, follow
the staged, check-mode-first [real-host validation runbook](docs/real-host-validation.md).

## License

GPL-2.0-or-later. See [LICENSE](LICENSE).
