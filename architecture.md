# Architecture

## System Overview

The `rhel-oe-build-lab` simulates a small Red Hat Operating
Environment under change management. One control node drives
configuration and patching across a fleet of managed hosts via SSH.
Git is the source of truth for all configuration, playbooks, and
change records.

```
+---------------------------+
| control-node              |
|  - Ansible control        |
|  - Git working tree       |
|  - Change records         |
+-------------+-------------+
              |
              | SSH (key-based)
              | become: sudo -> root
              |
   +----------+----------+--------------+
   |          |          |              |
+--v---+   +--v---+   +--v---+      +---v--+
|server|   |server|   |server|      |server|
|  a   |   |  b   |   |  c   |      |  d   |
+------+   +------+   +------+      +------+

All managed hosts run a RHEL-compatible OS (RHEL, Rocky, Alma)
and are members of the [production] inventory group.
```

---

## Components

| Component        | Purpose                                                   |
|------------------|-----------------------------------------------------------|
| control-node     | Runs Ansible, holds the Git working tree, executes change |
| servera–serverd  | Managed Linux hosts representing the production estate    |
| `inventory`      | Defines the production group and connection defaults      |
| `group_vars/`    | Shared and group-scoped variables                         |
| `roles/`         | Reusable configuration units (baseline, users, firewall…) |
| `playbooks/`     | Orchestrate roles into named operations                   |
| `change-records/`| Markdown change records (CRQ-NNN-*.md), one per change    |
| `docs/`          | Operational documentation (validation, troubleshooting)   |
| `screenshots/`   | Portfolio evidence captured during runs                   |

---

## Change Flow

A change moves through eight stages. Every stage has a paper trail.

1. Engineer edits a role or playbook on a Git branch.
2. `ansible-playbook --syntax-check` confirms YAML is well-formed.
3. `ansible-playbook --check --diff` previews the change.
4. Change record (CRQ) is drafted with reason, risk, rollback.
5. Playbook is applied against `production`. Output goes to
   `ansible.log` (project-scoped).
6. Validation playbook confirms expected end state.
7. Change record is updated with results and any deviations.
8. Branch is merged. Commit SHA or tag is recorded in the CRQ.

---

## Security Posture

| Control     | Target State                                          |
|-------------|-------------------------------------------------------|
| SELinux     | Enforcing on all managed hosts                        |
| firewalld   | Enabled, default zone `public`, minimum service set   |
| SSH         | Key-based only, root login disabled                   |
| Sudo        | Passwordless for `admin` group only, audited          |
| auditd      | Enabled, rules for privileged commands and identity   |
| chrony      | Enabled, time sync against trusted NTP sources        |
| Logging     | journald + rsyslog forwarding (planned in Lab 2)      |

### Current Deviation

The inventory currently connects as `root` for lab convenience.
The target state is a dedicated `admin` user created by the
`users` role, with `baseline_hardening` then disabling direct
root SSH. This transition will be performed as a tracked change
(future `CRQ-002`).

---

## What Could Go Wrong

| Risk                                | Mitigation                                |
|-------------------------------------|-------------------------------------------|
| Playbook breaks SSH access          | Always `--check --diff` first             |
| Update reboots a host mid-business  | Quarterly window, staggered targets       |
| SELinux denial after change         | `ausearch -m AVC -ts recent` documented   |
| Patch introduces regression         | Validation playbook + rollback checklist  |
| Drift between hosts                 | Baseline run before every quarterly cycle |

---

## RHCE Mapping

This architecture exercises the following RHCE objective areas:

- Install and configure an Ansible control node
- Configure managed nodes for Ansible administration
- Run playbooks; use `--check`, `--diff`, `--syntax-check`
- Write roles and import them from playbooks
- Use variables, conditionals, loops, handlers
- Manage SELinux, firewalld, sshd, sudo, chrony via modules
- Manage parallelism (forks)
- Troubleshoot playbook failures with verbose mode and facts
