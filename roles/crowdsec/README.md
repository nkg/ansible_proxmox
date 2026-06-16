# nkg.proxmox.crowdsec

Install and configure [CrowdSec](https://www.crowdsec.net/) on a Proxmox host:
the detection engine and nftables/iptables firewall bouncer (from the upstream
GitHub release tarballs), hub collections/scenarios/parsers/postoverflows,
acquisition + IP/CIDR whitelists, ban duration, Prometheus metrics, and
optional console enrollment.

## Requirements

A running Proxmox VE host (Debian). Run with `become`.

## Role Variables

See `defaults/main.yml` and `meta/argument_specs.yml`.

| Variable | Default | Description |
|----------|---------|-------------|
| `crowdsec_version` | `1.7.6` | Engine version. |
| `crowdsec_bouncer_version` | `0.0.34` | Firewall bouncer version. |
| `crowdsec_firewall` | `nftables` | Bouncer backend. |
| `crowdsec_collections` | see defaults | Hub collections. |
| `crowdsec_whitelist_ips` | `[]` | Trusted IPs. |
| `crowdsec_whitelist_cidrs` | RFC1918 + Tailscale | Trusted CIDRs. |
| `crowdsec_console_enrollment_key` | `""` | Console enrollment key. |
| `crowdsec_skip_install` | `false` | Skip binaries (CI/molecule). |

## Example Playbook

```yaml
- hosts: proxmox
  become: true
  roles:
    - role: nkg.proxmox.crowdsec
      vars:
        crowdsec_whitelist_ips:
          - 203.0.113.10
```

## Status

Ported from `Regularmusic/iac` `crowdsec`. The example whitelist IP and
Hetzner-LAN CIDR from the source have been dropped from the defaults — set
`crowdsec_whitelist_ips` / `crowdsec_whitelist_cidrs` for your environment.
