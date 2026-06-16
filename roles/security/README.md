# nkg.proxmox.security

Applies the externally-maintained
[konstruktoid.hardening](https://github.com/konstruktoid/ansible-role-hardening)
CIS/STIG baseline to a Proxmox host, with a vetted set of **Proxmox-safe
overrides**. konstruktoid's defaults are deliberately aggressive and will break
a Proxmox node (UFW vs. bridges, root PAM login removed, SSH host keys
regenerated, auditd version conflicts on Trixie, etc.) — this role encodes the
overrides that keep the host working while still hardening it.

## Why a wrapper, not a hand-rolled role

konstruktoid.hardening is actively maintained (frequent releases, Debian 12/13
+ Ubuntu support) and covers far more than a bespoke ssh/fail2ban/auditd role
would. The reusable value here is the *curated Proxmox configuration*, not a
re-implementation of hardening.

## Requirements

- A running Proxmox VE host (Debian). Run with `become`.
- The `konstruktoid.hardening` role installed. It is a standalone Galaxy role
  (not a collection), so it can't be a `galaxy.yml` dependency — install it via
  the collection's `requirements.yml`:

  ```bash
  ansible-galaxy role install -r requirements.yml
  ```

- The companion fixes for konstruktoid's side effects live in
  `nkg.proxmox.host_config`: enable `host_config_lxc_apparmor_fix` and
  `host_config_tmp_exec_override` when running this role, or LXC management and
  Terraform remote-exec will break.

## Role Variables

Every konstruktoid setting this role tunes is exposed as a `security_*`
variable — see `defaults/main.yml` and `meta/argument_specs.yml`. Highlights:

| Variable | Default | Purpose |
|----------|---------|---------|
| `security_apply_hardening` | `true` | Master on/off switch. |
| `security_ssh_port` | `22` | SSH port to allow. |
| `security_manage_ufw` | `false` | UFW off (bridge conflict). |
| `security_disable_root_account` | `false` | Keep root PAM for the GUI. |
| `security_manage_auditd` | `false` | Off (Trixie version conflict). |
| `security_pwquality` | relaxed | Password policy (key-only SSH). |
| `security_packages_debian` | Trixie-safe | Package list override. |

## Example Playbook

```yaml
- hosts: proxmox
  become: true
  roles:
    - role: nkg.proxmox.host_config
      vars:
        host_config_lxc_apparmor_fix: true
        host_config_tmp_exec_override: true
    - role: nkg.proxmox.security
    - role: nkg.proxmox.crowdsec
```

## Status

Replaces the earlier stub role. Override set derived from `Regularmusic/iac`
group_vars (which runs konstruktoid.hardening v4.x on the Hetzner pve host).
