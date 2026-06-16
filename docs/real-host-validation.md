# Real-host validation runbook

The molecule tests only exercise config-level behaviour in containers. This
runbook validates the roles against a real Proxmox node, in the order that
actually works: **bootstrap the IaC user first, then run everything as it.**

> **Do NOT** run `nkg.proxmox.bootstrap_hetzner` / the `hetzner_image` role
> against a host you care about — it **wipes disks**. Out of scope here.

## 0. Prerequisites

- A **non-critical, rebuildable** Proxmox node, and a **console/KVM session**
  open (the security stage changes sshd/PAM).
- The collection + konstruktoid role installed (or a local checkout symlinked
  into `~/.ansible/collections/ansible_collections/nkg/proxmox` for dev):
  ```bash
  ansible-galaxy collection install git+https://github.com/nkg/ansible_proxmox.git,v1 --force
  ansible-galaxy role install -r <(curl -fsSL https://raw.githubusercontent.com/nkg/ansible_proxmox/main/requirements.yml)
  ```
- An inventory with a `proxmox` group. If root SSH login is disabled, you bootstrap
  via an existing unprivileged login (`ng` below) escalating with `su`.

## 1. Bootstrap the IaC user (FIRST)

This creates the `iac` Linux user (passwordless sudo + your SSH key) **and** the
Proxmox API user/role/token. Connect as your existing login and escalate via
`su` (root's password at the `BECOME` prompt). Pass your public key by **file
path** (not `$(cat …)` — shell substitution can mangle it):

```bash
ansible-playbook nkg.proxmox.api_access -l <node> \
  -e ansible_user=ng -e ansible_become_method=su -K \
  -e api_access_admin_ssh_key_file=/absolute/path/to/your_key.pub
```

The playbook puts `/usr/sbin` on PATH for every task, so the Proxmox CLI
(`pveum`/`pvesh`) resolves even under a non-login `su`. If you only want the
Linux user (not the API token yet), add `--tags bootstrap`.

Confirm it worked: `ssh iac@<node-ip> true` should succeed by key.

## 2. Everything else runs as `iac`

`iac` has key auth + passwordless sudo, so `sudo`'s `secure_path` includes
`/sbin` — no more `su`/PATH issues. Drop `-K`, `su`, and `become_flags`:

```bash
ansible-playbook nkg.proxmox.site -l <node> -e ansible_user=iac \
  -e host_base_system_upgrade=false --tags repos,host_base,host_config
```

`-e ansible_user=iac` is required if your inventory hardcodes `ansible_user:
root` (inventory vars outrank `-u`; only `-e` wins).

Verify on the node:
```bash
apt update                               # succeeds (enterprise repo disabled)
cat /etc/apt/sources.list.d/pve-no-subscription.sources
systemctl is-enabled pve-ha-lrm          # disabled
timedatectl
```

## 3. Security (konstruktoid) — on a snapshot

The heavy stage: module blacklists, GRUB cmdline, sysctl, PAM, sshd. **Snapshot
first.**

```bash
ansible-playbook nkg.proxmox.site -l <node> -e ansible_user=iac --tags security
```

Then, **before closing your session**, in a NEW shell confirm SSH still works,
and that an unprivileged LXC still starts (`pct start <ctid>`) — that proves the
user-namespace sysctl override held.

## 4. Idempotence check (the real proof)

```bash
ansible-playbook nkg.proxmox.site -l <node> -e ansible_user=iac -e host_base_system_upgrade=false
```
Expect `changed=0` on the recap. Any `changed` on a steady-state re-run is a bug.

## Real-world findings (and how the collection handles them)

Things that surfaced validating against real Proxmox 9 (Trixie) nodes:

| Symptom | Cause | Handled by |
|---------|-------|-----------|
| `apt update` 401 on `enterprise.proxmox.com` | stock enterprise/ceph repos enabled, no subscription | `repos` disables the stock `pve-enterprise`/`ceph` `.sources` first (file module, before any apt) |
| `apparmor_parser` / `pveum`: `No such file or directory` | `/usr/sbin` not on PATH under non-login `su` | play-level `environment: PATH` (incl. `/usr/sbin`) in `site.yml`/`api_access.yml`; or run as `iac` (sudo `secure_path`) |
| `Configure RuleFile … usbguard-daemon.conf does not exist` | konstruktoid USBGuard on a hypervisor (and `--check` can't install it) | `security_manage_usbguard: false` by default |
| unprivileged LXC won't start after hardening | konstruktoid sets `user.max_user_namespaces=0` | `security_sysctl_overrides` re-asserts userns after hardening |
| `authorized_key: list index out of range` | mangled/empty key string (fish `$(cat …)`) | pass `api_access_admin_ssh_key_file=<path>`; role reads + trims it |
| `community.general.yaml … deprecated` / `AnsibleDumper` callback error | deprecated yaml stdout callback | set `stdout_callback = default` + `callback_result_format = yaml` in your `ansible.cfg` |

## Connection / become notes

- **Inventory `ansible_user` outranks `-u`** — override with `-e ansible_user=…`.
- **`su` (non-login)** gives a PATH without `/sbin`. Use `iac` + sudo (preferred),
  or `-e ansible_become_flags='-'` for a login shell.
- The IaC user that `api_access` creates gets passwordless sudo, so after step 1
  you never need `su`/`-K` again.
