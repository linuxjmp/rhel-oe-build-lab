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
| Deploys SSH drop-in config | Controls `PermitRootLogin` and `PasswordAuthentication` |

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `baseline_packages` | `[chrony, audit, firewalld, policycoreutils-python-utils]` | Packages to install |
| `baseline_selinux_state` | `enforcing` | SELinux mode |
| `baseline_firewalld_default_zone` | `public` | firewalld default zone |
| `baseline_firewalld_services` | `[ssh]` | Services allowed at baseline |
| `baseline_sshd_permit_root_login` | `yes` | Root SSH login (see note below) |
| `baseline_sshd_password_authentication` | `no` | SSH password auth |
| `baseline_motd_enabled` | `true` | Deploy MOTD login banner to `/etc/motd` |

### Root Login Default

`baseline_sshd_permit_root_login` defaults to `yes` because the lab
inventory currently connects as `root`. Setting it to `no` before the
`users` role has created an admin account with SSH keys will lock you
out of the managed hosts.

**Safe migration path (tracked as CRQ-002):**
1. Run the `users` role to create an admin account with your SSH key.
2. Test SSH login as the admin user from the control node.
3. Set `baseline_sshd_permit_root_login: "no"` in `group_vars/all.yml`.
4. Re-apply the baseline. Root SSH is now disabled.

## Handlers

| Handler | Trigger |
|---------|---------|
| `Restart sshd` | SSH drop-in config changes |

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
| `ssh` | SSH drop-in config |

```bash
# Example: re-apply only the SSH config
ansible-playbook playbooks/01-baseline.yml --tags ssh
```

## Relationship to Other Roles

This role ensures services are **running**. The dedicated roles
extend and specialize:

- `firewall` — manages which services/ports are allowed
- `auditd` — deploys the audit rule set
- `users` — creates accounts and manages the admin sudoers drop-in

All roles are idempotent and can be re-applied without conflicts.
