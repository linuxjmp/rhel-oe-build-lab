# Role: firewall

Declarative firewalld management for the `rhel-oe-build-lab` fleet.
Installs firewalld, ensures it is running, and manages which services
and ports are allowed in the default zone.

## What This Role Does

| Task | Effect |
|------|--------|
| Installs `firewalld` package | Ensures the daemon is present |
| Starts and enables `firewalld` | Survives reboots |
| Sets the default zone | Standardizes zone across the fleet |
| Enables services from `firewall_services` | Persistent + immediate |
| Enables ports from `firewall_ports` | Persistent + immediate |
| Removes services in `firewall_remove_services` | Persistent + immediate |
| Removes ports in `firewall_remove_ports` | Persistent + immediate |

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `firewall_default_zone` | `public` | Zone to manage |
| `firewall_services` | `[ssh]` | Named services to allow |
| `firewall_ports` | `[]` | Explicit port/proto pairs to allow |
| `firewall_remove_services` | `[]` | Services to explicitly block |
| `firewall_remove_ports` | `[]` | Ports to explicitly block |

## Example: Allow an application port

```yaml
# group_vars/all.yml or host_vars/servera.yml
firewall_ports:
  - "8443/tcp"

firewall_services:
  - ssh
  - cockpit
```

## Relationship to baseline_hardening

`baseline_hardening` ensures firewalld is started and adds `ssh` to
the default zone as part of the host's foundational security posture.
This role extends that with a fully declarative rule set.

When both roles run in the same play (as in `01-baseline.yml`), the
firewall role's tasks are additive and idempotent — no conflicts occur.

## Manual Verification

```bash
firewall-cmd --get-default-zone
firewall-cmd --list-services
firewall-cmd --list-ports
firewall-cmd --list-all
```

## Tags

All tasks use the `firewall` tag.

```bash
ansible-playbook playbooks/01-baseline.yml --tags firewall
```
