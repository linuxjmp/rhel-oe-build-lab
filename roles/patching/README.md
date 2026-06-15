# Role: patching

Quarterly patching workflow for the `rhel-oe-build-lab` fleet.
Takes pre/post snapshots, applies updates, checks for reboot
requirements, and publishes results as host facts for report
templates.

## What This Role Does

1. Captures pre-update kernel, package list, and service states.
2. Refreshes the dnf metadata cache.
3. Applies security-only or full updates, depending on `patching_update_mode`.
4. Checks whether a reboot is needed (`dnf needs-restarting -r`).
5. Optionally reboots (if `patching_allow_reboot: true`).
6. Captures post-update kernel, package list, and service states.
7. Publishes all results as `patching_*` host facts.

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `patching_update_mode` | `full` | `full` (all updates) or `security` (errata only) |
| `patching_allow_reboot` | `false` | Reboot automatically when needed |
| `patching_reboot_timeout` | `600` | Seconds to wait for host after reboot |
| `patching_baseline_services` | `[firewalld, auditd, chronyd, sshd]` | Services to snapshot |

## Published Facts

After the role runs, the following facts are available on the host
and accessible from `delegate_to: localhost` tasks via `hostvars`:

| Fact | Type | Description |
|------|------|-------------|
| `patching_pre_kernel` | string | Kernel version before patching |
| `patching_post_kernel` | string | Kernel version after patching |
| `patching_pre_packages` | list | Package list before (rpm -qa sorted) |
| `patching_post_packages` | list | Package list after (rpm -qa sorted) |
| `patching_pre_services` | string | Service states before |
| `patching_post_services` | string | Service states after |
| `patching_update_result` | dict | Raw dnf module result |
| `patching_reboot_required` | bool | Whether a reboot is needed |

## Typical Usage

In `02-quarterly-update.yml`:

```yaml
- name: Quarterly OE update
  hosts: production
  serial: 1
  tasks:
    - name: Apply patches
      ansible.builtin.import_role:
        name: patching

    - name: Render per-host report
      ansible.builtin.template:
        src: quarterly_report.md.j2
        dest: "reports/{{ inventory_hostname }}.md"
      delegate_to: localhost
```

## Reboot Behavior

By default (`patching_allow_reboot: false`), the role reports that
a reboot is needed but does not perform one. The operator reboots
manually and then re-runs the validation playbook.

Set `patching_allow_reboot: true` in a group var or on the CLI:

```bash
ansible-playbook playbooks/02-quarterly-update.yml \
  -e patching_allow_reboot=true
```

## Tags

| Tag | Tasks covered |
|-----|--------------|
| `patching` | All tasks |
| `snapshot` | Pre/post snapshot capture |
| `update` | dnf update tasks |
| `reboot` | Reboot check and optional reboot |
