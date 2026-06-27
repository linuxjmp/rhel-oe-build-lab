# Scheduling

This document covers the static `systemd/` units that schedule the safe,
non-destructive parts of the quarterly workflow. The design rule is
simple: **schedule detection and preview, require a human to approve
remediation.**

> Related: `playbooks/05-schedule-validation.yml` (CRQ-005) installs an
> Ansible-managed version of the weekly drift check by rendering the
> templates under `scheduling/`. The static units here are a portable,
> copy-deploy alternative for a control node where you'd rather drop the
> files straight under `/etc/systemd/system/` and also add the quarterly
> dry-run. Pick one mechanism for weekly validation — don't enable both
> timers at once or the fleet gets validated twice on different days.

## Why full patching is not blindly scheduled

Patching mutates running production systems: it replaces packages,
restarts services, and sometimes reboots. A patch that fixes one host
can break another (a kernel that won't boot, a config rewritten by an
RPM, a service that won't come back). An unattended cron job that
applies updates fleet-wide has no judgement and no brakes — by the time
anyone notices, every host is already changed.

Change control exists precisely so a human reviews *what* will change,
applies it to a canary first, validates, and only then rolls forward.
Automating that decision away throws out the part that makes it safe.

So this lab schedules only the actions that cannot hurt production:
- a **read-only validation** run, and
- a **dry-run preview** of the quarterly update.

The actual patch deployment stays a deliberate, human-approved step.

## Validation scheduling vs. update scheduling

| | Weekly validation | Quarterly update |
|---|---|---|
| Playbook | `03-validation.yml` | `02-quarterly-update.yml` |
| Scheduled as | the real playbook | `--check --diff` (dry-run only) |
| Changes anything? | No — asserts state only | Scheduled run: no. Real run: yes (manual) |
| Scope | full fleet | canary host (`servera`) for the scheduled dry-run |
| Safe unattended? | Yes | Only the dry-run; the real apply is manual |

Validation is **detection**. The quarterly update is **remediation**.
Keeping them on separate units, separate schedules, and separate
approval requirements is the whole point — you never want drift
detection to quietly trigger a patch.

## Why the quarterly update is scheduled as `--check --diff`

`--check` puts Ansible in dry-run mode: tasks report what they *would*
do without doing it. `--diff` shows the specific before/after changes.
Together they produce a standing preview of the next patch cycle —
"these packages would update, this service would restart" — captured in
the journal ahead of the change window, with zero production impact.

The scheduled timer therefore gives you change *visibility* on a
cadence, while leaving the change *decision* to a human.

## Why `servera` is used as a canary host

`servera` is the first host in the `[production]` group and is treated
as the canary. Scoping the scheduled dry-run with `--limit servera`
keeps even the read-only run small and focused, and it mirrors the
manual rollout pattern: prove a change on one host, validate it, then
widen to the fleet. The canary limits blast radius — a problem shows up
on one host before it can reach four.

## How to install the systemd units

Copy the unit files onto the control node and reload systemd. The units
assume the repo is deployed at `/opt/rhel-oe-build-lab` (the
`WorkingDirectory=` in each service). Adjust that path in both
`.service` files if you deploy elsewhere.

```bash
sudo cp systemd/*.service systemd/*.timer /etc/systemd/system/
sudo systemctl daemon-reload
```

## How to enable the timers

```bash
sudo systemctl enable --now rhel-oe-weekly-validation.timer
sudo systemctl enable --now rhel-oe-quarterly-dryrun.timer
```

`--now` starts the timer immediately as well as enabling it on boot.
You are enabling the **timers**, not the services — the timers trigger
the oneshot services on schedule.

## How to check timer status

```bash
# Next/last run for both timers
systemctl list-timers | grep rhel-oe

# Detailed state of a single timer
systemctl status rhel-oe-weekly-validation.timer
systemctl status rhel-oe-quarterly-dryrun.timer
```

## How to read logs with journalctl

Each scheduled run writes its full output to the journal under the
service name:

```bash
journalctl -u rhel-oe-weekly-validation.service
journalctl -u rhel-oe-quarterly-dryrun.service

# Just the most recent run, following live:
journalctl -u rhel-oe-quarterly-dryrun.service -n 200 --no-pager
```

A failed weekly validation run leaves the service in the `failed`
state, so `systemctl --failed` surfaces drift without reading logs.

## How to manually approve and run the real quarterly update

The scheduled dry-run is the input to a human decision. When a change
window opens, review the preview, then apply for real — one host, then
the fleet:

```bash
# 1. Review the latest dry-run output
journalctl -u rhel-oe-quarterly-dryrun.service

# 2. Apply to the canary
ansible-playbook playbooks/02-quarterly-update.yml --limit servera

# 3. Validate the canary
ansible-playbook playbooks/03-validation.yml --limit servera

# 4. Apply to the rest of the fleet
ansible-playbook playbooks/02-quarterly-update.yml

# 5. Validate the fleet
ansible-playbook playbooks/03-validation.yml
```

Step 2 onward is intentionally run by hand. `02-quarterly-update.yml`
already patches `serial: 1` with an inline post-patch health gate
(CRQ-003), so a degraded host halts the rollout before the next host is
touched.

## How to validate after patching

Always run `03-validation.yml` after applying updates — on the canary
before widening, and on the full fleet after. A clean (exit 0) run is
your evidence that the patched hosts still match the expected baseline.
Save the output as portfolio/change evidence:

```bash
ansible-playbook playbooks/03-validation.yml | tee reports/validation-$(date +%Y%m%dT%H%M%S).txt
```

## How rollback fits into the process

If validation fails after a patch — a service won't start, a check
asserts false, a host won't come back cleanly — stop the rollout and
roll the affected host back rather than pushing forward. The canary-first
ordering means most failures are caught on `servera` before the fleet is
touched.

```bash
# Guided, single-host rollback (prompts for the dnf transaction ID
# and a confirmation word):
ansible-playbook playbooks/04-rollback-checklist.yml --limit servera
```

See `playbooks/04-rollback-checklist.yml` and the README's "Running the
Rollback Checklist" section for the full procedure. Rollback is the
safety net that makes scheduling the dry-run — and approving the real
patch by hand — a low-risk operation.
