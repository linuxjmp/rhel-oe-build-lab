# CRQ-005: Weekly Scheduled Drift Detection

| Field           | Value                                                              |
|-----------------|--------------------------------------------------------------------|
| Change ID       | CRQ-005                                                            |
| Title           | systemd timer to run 03-validation.yml weekly on the control node  |
| Owner           | jaric.poinsette                                                   |
| Date submitted  | 2026-06-17                                                        |
| Risk level      | Low (control-node only; read-only against the fleet)             |
| Status          | Implemented — pending control-node install                       |

---

## 1. Reason for Change

The validation playbook (`03-validation.yml`) only proves posture at
the moment someone runs it — after a baseline apply or a quarterly
patch. Between those events a host can drift: someone sets SELinux
permissive for a quick test and forgets, a service gets disabled, a
sudoers drop-in is edited by hand. Nothing catches it until the next
manual run.

This change schedules the existing validation playbook to run weekly
so drift is surfaced within days, not at the next quarter.

## 2. What Changed

New control-node scheduling artifacts (this is infrastructure on the
control node — it does **not** modify managed hosts):

- `scheduling/rhel-oe-validation.sh.j2` — wrapper that runs
  `03-validation.yml`, tees a timestamped log to `reports/`, and exits
  non-zero on drift so the systemd unit reports `failed`.
- `scheduling/rhel-oe-validation.service.j2` — `oneshot` unit.
- `scheduling/rhel-oe-validation.timer.j2` — `OnCalendar=Mon *-*-*
  06:00:00`, `Persistent=true`, `RandomizedDelaySec=15m`.
- `playbooks/05-schedule-validation.yml` — `hosts: localhost` installer
  that renders the templates into `/usr/local/bin` and
  `/etc/systemd/system`, reloads systemd, and enables the timer.
  `-e schedule_state=absent` cleanly removes it.
- `scheduling/README.md` — operator notes (incl. why timer over cron).

## 3. Validation

- `ansible-playbook playbooks/05-schedule-validation.yml --syntax-check`
  — pass.
- Pending: install on the control node
  (`ansible-playbook playbooks/05-schedule-validation.yml
  --ask-become-pass`), confirm with `systemctl list-timers
  rhel-oe-validation.timer`, and trigger one on-demand run
  (`systemctl start rhel-oe-validation.service`) to verify the log
  lands in `reports/` and a clean fleet exits 0.

## 4. Rollback

`ansible-playbook playbooks/05-schedule-validation.yml
-e schedule_state=absent --ask-become-pass` stops/disables the timer
and removes the units and wrapper, then reverts this commit.

## 5. Follow-up

Add an `OnFailure=` hook so a failed weekly check notifies (email or a
webhook into the monitoring lab) instead of waiting to be noticed in
`systemctl --failed`.
