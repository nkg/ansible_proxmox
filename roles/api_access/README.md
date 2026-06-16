# nkg.proxmox.api_access

Create an IaC admin user and a Proxmox API token (for Terraform/OpenTofu) on an
already-running Proxmox VE host, ported from `Regularmusic/iac`
`proxmox_api_setup`. The role:

- creates a Linux admin user (dedicated group, `sudo` membership, passwordless
  sudoers drop-in, optional admin/root SSH keys);
- optionally sets content types on named storage pools (idempotent; skipped
  when `api_access_storage_pools` is empty);
- creates the Proxmox PAM user, group, role and `/` ACL;
- generates an `apitoken` API token, exposes it via the `api_access_token`
  fact, and optionally writes the secret to a local file.

## Requirements

A running Proxmox VE host (Debian). Run with `become`. Requires the
`ansible.posix` collection (for `ansible.posix.authorized_key`).

> Web-UI TLS is **not** handled by this role. Use the ACME tasks in the
> `nkg.proxmox.host_config` role for a trusted certificate on the Proxmox UI.

## Role Variables

See `defaults/main.yml` and `meta/argument_specs.yml`.

| Variable | Default | Description |
|----------|---------|-------------|
| `api_access_user` | `iac` | Linux + Proxmox PAM username (group/role/ACL derived from it). |
| `api_access_role_privileges` | Terraform-capable set | Privileges granted to the IaC Proxmox role. |
| `api_access_admin_ssh_key` | `""` | SSH public key for the admin user (empty skips). |
| `api_access_root_ssh_key` | `""` | SSH public key for root (empty skips). |
| `api_access_storage_pools` | `{}` | `{pool: "content,csv"}`; empty skips the storage step. |
| `api_access_token_output_path` | `""` | Controller path to write the token secret; empty only sets the fact. |

## API token handling

The generated token secret is registered, kept under `no_log`, and exposed as
the `api_access_token` fact (with `api_access_token_id`). Only the token **id**
is printed in debug output — never the secret. Set
`api_access_token_output_path` to also persist the secret to a local file
(mode `0600`). This role does **not** call SOPS; persist/encrypt the token in
your own pipeline if needed.

## Example Playbook

```yaml
- hosts: proxmox
  become: true
  roles:
    - role: nkg.proxmox.api_access
      vars:
        api_access_user: iac
        api_access_admin_ssh_key: "{{ vault_iac_ssh_public_key }}"
        api_access_storage_pools:
          local: "iso,vztmpl,snippets,backup,import"
        api_access_token_output_path: ./proxmox_api_token.secret
```

## Status

Ported from `Regularmusic/iac` `proxmox_api_setup`. The original "Save token to
SOPS" block and the `proxmox_acme.yml` import have been removed: token handling
is now controller-local (fact + optional file), and web-UI TLS belongs to the
`nkg.proxmox.host_config` ACME tasks. `api_access_user` and
`api_access_role_privileges` defaults come from that repo's
`group_vars/all.yml` (`iac_user` and `proxmox_admin_role_privileges`).
