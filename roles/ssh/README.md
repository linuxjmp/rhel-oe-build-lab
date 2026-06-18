# ssh role

Manages the sshd posture for the managed fleet through a single
validated drop-in at `/etc/ssh/sshd_config.d/00-baseline.conf`.

This concern used to live inside `baseline_hardening`. It was split
out under **CRQ-004** so SSH access policy — the control most likely
to lock you out of a host — can be read, changed, and validated on
its own without touching the rest of the baseline.

## What it does

- Renders `templates/sshd_baseline.conf.j2` to the drop-in path.
- Validates the rendered config with `sshd -t -f` **before** writing,
  so an invalid config never lands on a host.
- Restarts `sshd` (handler) only when the file changes.

## Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `ssh_permit_root_login` | `"yes"` | `PermitRootLogin` value. Set to `"no"` in `group_vars/all.yml` once the admin account is verified (CRQ-002). |
| `ssh_password_authentication` | `"no"` | Key-based auth only. |
| `ssh_dropin_path` | `/etc/ssh/sshd_config.d/00-baseline.conf` | Where the managed drop-in is written. |

## Ordering

In `playbooks/01-baseline.yml` this role runs **after** the `users`
role. The admin account and its key must exist before this role can
safely set `PermitRootLogin no`.

```
baseline_hardening → users → ssh → firewall → auditd
```
