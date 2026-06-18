# Role: baseline_hardening

Applies the foundational security and operational baseline to every
managed host. This role runs first in the `01-baseline.yml` playbook;
all other roles build on top of the state it establishes.

## What This Role Does

| Task | Effect |
|------|--------|
| Installs baseline packages | `chrony`, `audit`, `firewalld`, `policycoreutils-python-utils` |
| Sets SELinux to `enforcing` | Persistent across reboots |
| Starts and enables `firewalld` | Sets default zone; allows `ssh` |
| Starts and enables `auditd` | Audit daemon is running before rules load |
| Starts and enables `chronyd` | Time sync operational |
| Deploys MOTD login banner | `/etc/motd` showing host name and managed-system notice |
| Deploys `wheel-nopass` sudoers | Passwordless sudo for `wheel` group |

> **SSH posture moved out (CRQ-004):** `PermitRootLogin` and
> `PasswordAuthentication` are now managed by the dedicated `ssh`
> role. See `roles/ssh/`.

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `baseline_packages` | `[chrony, audit, firewalld, policycoreutils-python-utils]` | Packages to install |
| `baseline_selinux_state` | `enforcing` | SELinux mode |
| `baseline_firewalld_default_zone` | `public` | firewalld default zone |
| `baseline_firewalld_services` | `[ssh]` | Services allowed at baseline |
| `baseline_motd_enabled` | `true` | Deploy MOTD login banner to `/etc/motd` |

## Handlers

This role currently defines no handlers. (`Restart sshd` moved to the
`ssh` role with the sshd drop-in.)

## Tags

| Tag | Tasks covered |
|-----|--------------|
| `packages` | Package installation |
| `selinux` | SELinux state |
| `firewall` | firewalld service and default zone |
| `auditd` | auditd service |
| `chrony` | chronyd service |
| `motd` | MOTD login banner |
| `sudo` | wheel sudoers drop-in |

## Relationship to Other Roles

This role ensures services are **running**. The dedicated roles
extend and specialize:

- `ssh` — manages sshd posture (root login, password auth)
- `firewall` — manages which services/ports are allowed
- `auditd` — deploys the audit rule set
- `users` — creates accounts and manages the admin sudoers drop-in

All roles are idempotent and can be re-applied without conflicts.
