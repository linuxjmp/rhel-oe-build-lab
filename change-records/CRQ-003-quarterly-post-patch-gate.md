# CRQ-003: Post-Patch Validation Gate for the Quarterly Rollout

| Field           | Value                                                              |
|-----------------|--------------------------------------------------------------------|
| Change ID       | CRQ-003                                                            |
| Title           | Inline post-patch health gate in 02-quarterly-update.yml           |
| Owner           | jaric.poinsette                                                   |
| Date submitted  | 2026-06-17                                                        |
| Risk level      | Medium                                                            |
| Status          | Implemented — pending fleet validation                            |

---

## 1. Reason for Change

`02-quarterly-update.yml` already rolls out one host at a time
(`serial: 1`, `max_fail_percentage: 0`). But "one at a time" only
protects the fleet if a broken host actually **fails the play** — and
patching can leave a host that Ansible considers "successful" while it
is operationally broken:

- a service that did not come back after a daemon restart or reboot,
- SELinux knocked to permissive by a policy package update,
- a unit left in the `failed` state,
- time sync degraded after a chrony update.

Without a gate, the rollout happily proceeds to `serverb`, `serverc`,
`serverd` and breaks the whole fleet before anyone looks at a report.

## 2. What Changed

Added an inline post-patch block to `playbooks/02-quarterly-update.yml`
that runs on each host immediately after the `patching` role and before
the play moves on:

1. Collect SELinux state, baseline service activity
   (`firewalld`, `auditd`, `chronyd`, `sshd`), failed systemd units,
   and chrony tracking — each with `failed_when: false`.
2. Assemble a single `quarterly_validation_failures` list.
3. Render the per-host report (now includes a **Post-Patch Validation
   Gate** section) — this runs even on failure so there is evidence.
4. `assert` that the failure list is empty. Because of `serial: 1` +
   `max_fail_percentage: 0`, a failed assert aborts the play and the
   remaining hosts are never patched.

Files touched:

- `playbooks/02-quarterly-update.yml` — gate tasks + `quarterly_gate_services` var
- `playbooks/templates/quarterly_report.md.j2` — gate result section

## 3. Validation

- `ansible-playbook playbooks/02-quarterly-update.yml --syntax-check` — pass.
- Gate expression unit-tested off-host with mocked register data for
  both a clean host (0 failures) and a degraded host (service down,
  failed unit, permissive SELinux, bad time sync → 4 failures).
- Pending: run against the canary (`--limit servera`) at the next
  quarterly cycle and confirm a deliberately stopped service halts the
  rollout and writes a FAIL report.

## 4. Rollback

The gate is additive. To bypass it for a single emergency run, target
everything except the gate: `--skip-tags gate`. To remove it
permanently, revert this commit.
