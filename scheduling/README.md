# scheduling/ — weekly drift detection

These artifacts install a **systemd timer on the control node** that
runs `playbooks/03-validation.yml` against the fleet once a week. If a
host has drifted from the baseline (a service stopped, SELinux flipped
to permissive, a sudoers drop-in went missing, time sync degraded),
the validation playbook's asserts fail, the run exits non-zero, and the
systemd service lands in the `failed` state where it can be alerted on.

Tracked as **CRQ-005**.

## Why a timer instead of cron

- `Persistent=true` re-runs a check that was missed while the control
  node was powered off — a cron job would just silently skip it.
- Each run gets a proper journal entry (`journalctl -u
  rhel-oe-validation.service`) and a unit state you can monitor.
- `RandomizedDelaySec` avoids a thundering herd if this pattern is
  later cloned across multiple control nodes.

## Files

| File | Installed to | Purpose |
|------|--------------|---------|
| `rhel-oe-validation.sh.j2` | `/usr/local/bin/rhel-oe-validation.sh` | Runs `03-validation.yml`, tees a timestamped log to `reports/`, exits non-zero on drift. |
| `rhel-oe-validation.service.j2` | `/etc/systemd/system/rhel-oe-validation.service` | `oneshot` unit that calls the wrapper. |
| `rhel-oe-validation.timer.j2` | `/etc/systemd/system/rhel-oe-validation.timer` | `OnCalendar=Mon *-*-* 06:00:00`, persistent. |

These are Jinja templates rendered by `playbooks/05-schedule-validation.yml`
so the repo path and run-as user are filled in at install time.

## Install / inspect / remove

```bash
# Install (needs sudo on the control node)
ansible-playbook playbooks/05-schedule-validation.yml --ask-become-pass

# Confirm the next run
systemctl list-timers rhel-oe-validation.timer

# Run an on-demand drift check now
sudo systemctl start rhel-oe-validation.service
journalctl -u rhel-oe-validation.service --no-pager

# Remove the schedule
ansible-playbook playbooks/05-schedule-validation.yml \
  -e schedule_state=absent --ask-become-pass
```

## Wiring up alerting (optional, next step)

Add an `OnFailure=` drop-in to the service that triggers a notifier
unit (email, webhook to the monitoring lab, etc.) so a failed weekly
check pages someone instead of sitting quietly in `systemctl --failed`.
