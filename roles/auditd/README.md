# Role: auditd

Configures `auditd` and deploys a lab-appropriate set of audit rules
covering identity changes, privilege escalation, SSH config changes,
time changes, hostname changes, and failed access attempts.

## What This Role Does

| Task | Effect |
|------|--------|
| Installs `audit` package | Ensures auditd is present |
| Starts and enables `auditd` | Survives reboots |
| Deploys `99-lab-audit.rules` | Declarative audit rule set |
| Runs `augenrules --load` (handler) | Activates rules without restart |

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `auditd_rules_file` | `/etc/audit/rules.d/99-lab-audit.rules` | Rule file path |
| `auditd_buffer_size` | `8192` | Kernel audit buffer (events) |
| `auditd_failure_mode` | `1` | 1=syslog, 2=panic on audit write failure |
| `auditd_lock_rules` | `false` | Lock rules after load (requires reboot to change) |

## Audit Rule Coverage

| Rule key | What is monitored |
|----------|-------------------|
| `identity` | `/etc/passwd`, `shadow`, `group`, `gshadow` |
| `sudoers` | `/etc/sudoers` and `/etc/sudoers.d/` |
| `privileged` | `sudo`, `su`, `passwd` command execution |
| `sshd_config` | `/etc/ssh/sshd_config` and `sshd_config.d/` |
| `time_change` | `adjtimex`, `settimeofday`, `clock_settime`, `/etc/localtime` |
| `hostname_change` | `sethostname`, `setdomainname` |
| `access_failure` | `-EACCES` and `-EPERM` on file operations by real users |

## Searching Audit Logs

```bash
# All events for a key
ausearch -k identity
ausearch -k privileged

# Events in the last hour
ausearch -k sudoers -ts recent

# AVC denials (SELinux)
ausearch -m AVC -ts recent

# Interpret event IDs
aureport --summary
aureport -au            # authentication report
```

## Handler Note

`auditd` cannot be restarted normally under systemd on RHEL 8+.
This role uses `augenrules --load` to compile and activate rule
files from `/etc/audit/rules.d/` without disrupting the audit queue.

## Tags

All tasks use the `auditd` tag.

```bash
ansible-playbook playbooks/01-baseline.yml --tags auditd
```
