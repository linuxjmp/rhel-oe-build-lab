# CRQ-001: Q1 Quarterly Operating Environment Update

| Field           | Value                                                |
|-----------------|------------------------------------------------------|
| Change ID       | CRQ-001                                              |
| Title           | Q1 Quarterly OE Update — baseline + patch + validate |
| Owner           | 00linux                                              |
| Date submitted  | 2026-06-04                                           |
| Scheduled start | 2026-06-06 09:00 (local)                             |
| Scheduled end   | 2026-06-06 12:00 (local)                             |
| Risk level      | Medium                                               |
| Status          | Closed                                               |

---

## 1. Reason for Change

Bring the `rhel-oe-build-lab` fleet (servera, serverb, serverc,
serverd) to the Q1 baseline. The change applies the
`baseline_hardening` role, installs all available errata via
`dnf update`, and confirms end state with the validation playbook.

This is the first formal run of the quarterly cycle; the goal is to
exercise the full change → validate → document workflow end-to-end.

---

## 2. Systems Affected

| Host    | Group       | Role profile  |
|---------|-------------|---------------|
| servera | production  | App profile 1 |
| serverb | production  | App profile 2 |
| serverc | production  | App profile 3 |
| serverd | production  | App profile 4 |

Out of scope: `control-node`.

---

## 3. Risk Assessment

| Risk                              | Likelihood | Impact | Mitigation                                |
|-----------------------------------|------------|--------|-------------------------------------------|
| SSH access lost after baseline    | Low        | High   | `--check --diff` first; second admin path |
| Kernel update prevents boot       | Low        | High   | Previous kernel kept; GRUB rollback path  |
| Service regression after patch    | Medium     | Medium | Per-host validation; staged rollout       |
| SELinux denial on patched binary  | Medium     | Low    | `ausearch` + `audit2allow` within window  |

---

## 4. Pre-Checks

Run on the control node and capture output:

```bash
ansible -i inventory production -m ping
ansible-playbook -i inventory playbooks/01-baseline.yml --syntax-check
ansible-playbook -i inventory playbooks/01-baseline.yml --check --diff
ansible -i inventory production -m shell -a 'getenforce; \
  systemctl is-active firewalld auditd chronyd sshd; uname -r'
ansible -i inventory production -m shell -a 'rpm -qa | sort' \
  > pre-pkgs.txt
```

Attach `pre-pkgs.txt` to this record.

---

## 5. Implementation Steps

1. Open the change window; notify watchers.
2. Apply baseline:
   ```bash
   ansible-playbook -i inventory playbooks/01-baseline.yml
   ```
3. Apply quarterly update, staggered one host at a time:
   ```bash
   ansible-playbook -i inventory playbooks/02-quarterly-update.yml \
     --limit servera
   # validate, then proceed to serverb, serverc, serverd
   ```
4. Reboot hosts where the update requires it.
5. Run validation across the full fleet:
   ```bash
   ansible-playbook -i inventory playbooks/03-validation.yml
   ```

---

## 6. Validation Steps

See [../docs/validation-guide.md](../docs/validation-guide.md) for
the full procedure. Minimum bar:

- All hosts at `Enforcing` SELinux.
- `firewalld`, `auditd`, `chronyd`, `sshd` all active.
- `chronyc tracking` reports `Leap status: Normal`.
- `rpm -qa` diff captured as `post-pkgs.txt`.
- Application smoke test passes per host.

---

## 7. Rollback Plan

If any host fails validation:

1. Stop further rollout to remaining hosts.
2. Reboot the affected host to the previous kernel from GRUB if
   needed.
3. `dnf history undo <id>` to revert the package transaction.
4. Re-run `playbooks/03-validation.yml --limit <host>`.
5. Record the regression below under "Lessons Learned".

Detailed rollback procedure: `playbooks/04-rollback-checklist.yml`.
That playbook enforces single-host targeting, requires typed
confirmation, undoes the specified dnf transaction, optionally
reboots, and prints a manual checklist for steps that cannot be
safely automated (data restore, GRUB kernel selection, stale
config).

---

## 8. Results

| Field            | Value                                                                           |
|------------------|---------------------------------------------------------------------------------|
| Actual start     | 2026-06-06 14:59 (local)                                                        |
| Actual end       | 2026-06-11 17:23 (local)                                                        |
| Outcome          | Success                                                                         |
| Hosts succeeded  | servera, serverb, serverc, serverd                                              |
| Hosts deferred   | none                                                                            |
| Attached output  | `reports/20260606T145955-{servera..serverd}.md`, `reports/validation-20260611.txt`, `reports/pre-pkgs.txt` |
| Git commit / tag | see git log                                                                     |

---

## 9. Lessons Learned

- **No errata were available this cycle.** All four hosts were already current; the quarterly update produced no package changes. The before/after snapshots in the per-host reports confirm the fleet was at the latest available state.
- **Validation caught a false positive on disk pressure.** The playbook's `df` check flagged `/mnt/repo` (a RHEL ISO mounted on `/dev/sr0`) at 100%, which is expected behavior for a read-only optical device — not actual disk pressure. Fixed by adding `-x iso9660` to the exclusion list in `playbooks/03-validation.yml`. Future CRQs should note whether any ISO mounts are in service when interpreting disk check results.
- **Serial rollout worked as designed.** The `serial: 1` + `max_fail_percentage: 0` configuration meant a failure on any host would have halted the playbook before touching the rest of the fleet.

---

## 10. References

- [README.md](../README.md)
- [architecture.md](../architecture.md)
- [docs/validation-guide.md](../docs/validation-guide.md)
- [docs/troubleshooting.md](../docs/troubleshooting.md)
