# Real-host validation runbook

The molecule tests only exercise config-level behaviour in containers. Before
trusting this collection, validate the **safe config roles** against a real
Proxmox node. This runbook uses a staged, check-mode-first approach.

> **Do NOT** run `nkg.proxmox.bootstrap_hetzner` / the `hetzner_image` role
> against any host you care about — it **wipes disks**. It is out of scope here.

## 1. Pick a node

Use a **non-critical, rebuildable** Proxmox node first. From the
`homeservers/platform` inventory the `proxmox` group is `vaterland`, `pvebu`,
`pvemini`, `linc` (`ansible_user: root`). Prefer a spare/test box, and take a
backup or snapshot of it before Stage 4.

Keep a **second root session / console (KVM/IPMI)** open throughout — Stage 4
(hardening) changes sshd and PAM, and you want a fallback if you lock yourself
out.

## 2. Install the collection + its role dependency

```bash
ansible-galaxy collection install git+https://github.com/nkg/ansible_proxmox.git
ansible-galaxy role install -r \
  <(curl -fsSL https://raw.githubusercontent.com/nkg/ansible_proxmox/main/requirements.yml)
```

Or, from `homeservers/platform`, add to `ansible/requirements.yml`:

```yaml
collections:
  - name: https://github.com/nkg/ansible_proxmox.git
    type: git
    version: main
roles:
  - src: https://github.com/konstruktoid/ansible-role-hardening/archive/refs/tags/v4.6.0.tar.gz
    name: konstruktoid.hardening
```

then `ansible-galaxy install -r ansible/requirements.yml`.

## 3. What runs by default

Environment-specific behaviour is **off** by default, so a plain run is safe:

| Role | Default behaviour | Off by default |
|------|-------------------|----------------|
| `repos` | enables pve-no-subscription, installs signing key, removes nag | enterprise, ceph |
| `host_base` | hostname/timezone/packages/sysctl/limits | — |
| `host_config` | disables single-node HA only | networking, CPU/SMT, storage, ACME |
| `crowdsec` | installs engine + firewall bouncer | console enrollment |
| `security` | applies konstruktoid with Proxmox-safe overrides | — |

## 4. Staged run

All commands assume you're in a tree whose `ansible.cfg` points at your
inventory (e.g. `homeservers/platform/ansible`). Replace `<node>`.

### Stage 0 — check mode (no changes)

```bash
ansible-playbook nkg.proxmox.site -l <node> --check --diff
```

Some `apt`/`command` tasks can't predict in check mode — a few yellow
"skipped"/"changed" lines are expected; you're looking for unexpected
**failures**, not a perfectly clean diff.

### Stage 1 — repos + host_base (lowest risk)

```bash
ansible-playbook nkg.proxmox.site -l <node> --tags repos,host_base
```

Verify on the node:

```bash
cat /etc/apt/sources.list.d/pve-no-subscription.sources
ls -l /etc/apt/keyrings/proxmox-release-*.gpg
apt update            # should succeed with the new deb822 source
timedatectl ; sysctl net.ipv4.ip_forward
```

### Stage 2 — host_config (just HA disable by default)

```bash
ansible-playbook nkg.proxmox.site -l <node> --tags host_config
systemctl is-enabled pve-ha-lrm pve-ha-crm   # expect: disabled
```

### Stage 3 — crowdsec

```bash
ansible-playbook nkg.proxmox.site -l <node> --tags crowdsec
cscli metrics ; cscli decisions list
systemctl status crowdsec crowdsec-firewall-bouncer --no-pager
```

### Stage 4 — security (konstruktoid) — RISKIEST, do last

```bash
ansible-playbook nkg.proxmox.site -l <node> --tags security,hardening
```

Then, **before closing your current session**, open a NEW ssh session to
confirm you can still log in, and check:

```bash
sshd -T | grep -Ei 'permitrootlogin|passwordauthentication'
aa-status | head
```

The role keeps root PAM login, disables UFW, and preserves SSH host keys
(Proxmox-safe overrides), but verify anyway.

## 5. Idempotence check (the real proof)

Re-run the full play and confirm the second pass reports **no changes**:

```bash
ansible-playbook nkg.proxmox.site -l <node>
# look for: changed=0 on the recap
```

Any task reporting `changed` on a steady-state re-run is a bug worth filing.

## 6. Rollback notes

- `host_config` backs up `/etc/network/interfaces` before rewriting it — but
  networking is **off by default**, so nothing is touched unless you enable
  `host_config_manage_networking`.
- `repos`: set `repos_no_subscription: false` to remove the `.sources` file.
- `security` (konstruktoid) makes broad changes — a node snapshot/backup before
  Stage 4 is the cleanest rollback.

## 7. Optional / advanced

- **ACME web-UI cert** (`host_config_acme_enabled: true`) needs a Cloudflare
  DNS-01 token (`host_config_acme_cf_token`) and `host_config_acme_email`.
- **`api_access`** (IaC user + API token) is a separate concern — run it on its
  own once the base validation passes; it needs `api_access_user` and SSH keys.
