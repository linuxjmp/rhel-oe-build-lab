# Lessons Learned

This document has two parts:

1. **Operational lessons** — specific findings from actual runs of this lab.
   Each entry explains what happened, why it matters, and what changed.

2. **Educational framing** — what this lab teaches, how it maps to real
   operations practice, and what a beginner should take away.

---

## What This Lab Teaches

This lab is a working model of a small enterprise RHEL operations
environment run under change management. It exercises the complete
lifecycle a Linux infrastructure engineer owns:

| Lifecycle phase | Lab artifact |
|-----------------|--------------|
| Configuration baseline | `01-baseline.yml` + five Ansible roles |
| Quarterly patch cycle | `02-quarterly-update.yml` + patching role |
| Post-change validation | `03-validation.yml` with assert-based PASS/FAIL |
| Rollback preparation | `04-rollback-checklist.yml` with confirmation gates |
| Change record | `change-records/CRQ-001-*.md` |
| Audit evidence | `reports/`, `screenshots/` |
| Documentation | `docs/`, role READMEs |

The key insight the lab demonstrates is that **automation is only part
of operations**. Tooling (Ansible) is paired with process (CRQ records),
evidence (reports), and correctness checks (validation). A playbook that
runs without errors but leaves a misconfigured host is not a success —
validation makes the difference.

---

## Mistakes Beginners Make

These are the most common patterns that trip up engineers new to
Ansible and RHEL operations. Each one appears in this lab either as a
deliberate design choice or as something that was fixed during a run.

### 1. Treating "the playbook ran" as "the change succeeded"

A clean `PLAY RECAP` with `failed=0` means the playbook ran to
completion — it does not mean the host is in the correct state.
Ansible tasks are idempotent by design; a task that was already correct
shows `ok` with no change. But that means a misconfiguration that
was already wrong before the run also shows `ok`.

**The fix:** run `03-validation.yml` after every change. The `assert`
tasks check actual state, not playbook success.

### 2. Locking yourself out with SSH changes

Applying `PasswordAuthentication no` or `PermitRootLogin no` before
an SSH key is in place on the host leaves no way back in without
console access. This lab explicitly defaults `PermitRootLogin: "yes"`
until a dedicated admin user is created (CRQ-002). The comment in
`roles/baseline_hardening/defaults/main.yml` explains the sequence.

**The fix:** plan the migration order — create admin user with SSH key,
test key login, then disable root SSH. Never skip step 2.

### 3. Writing tasks that are not idempotent

A task that reports `changed` every time it runs is hiding information.
If `changed` means "something was modified," then a task that is always
`changed` teaches you to ignore `changed` — which defeats the purpose.

Watch for: `command:` tasks without `changed_when`, shell scripts that
always output something, or `copy:` tasks that rewrite the file on
every run. This lab uses `changed_when: false` on all read-only
`command:` tasks.

### 4. Skipping `--check --diff` before applying

`--check` (dry run) + `--diff` (show what changes) is the only way to
know what a playbook will do before it does it. Skipping this step is
the Ansible equivalent of running a shell script without reviewing it.

This lab treats `syntax-check → check --diff → apply → validate` as
a non-negotiable sequence, not a suggestion.

### 5. Not capturing validation output

Running a validation playbook and closing the terminal is equivalent
to not running it. The audit trail requires the output. This lab
includes explicit commands to tee output to `reports/`.

### 6. Confusing "service enabled" with "service running"

`systemctl enable` sets the service to start at boot.
`systemctl start` starts it now.
An enabled-but-not-running service passes a `systemctl is-enabled`
check but fails a `systemctl is-active` check — and fails in
production when something tries to connect.

This lab always sets both `state: started` and `enabled: true`.

### 7. Writing disk pressure checks without testing on the actual fleet

The first run of `03-validation.yml` failed because `/mnt/repo` (an
ISO mount) reported 100% full. A read-only optical device always
reports 100%; that is not disk pressure. The fix was to add
`-x iso9660` to the `df` exclusion list — but only after running
`df -T` on the actual hosts first. See [CRQ-001 lessons](#crq-001--q1-quarterly-oe-update-2026-06-06) below.

---

## What Real Operations Teams Care About

This list is not exhaustive, but it reflects what an operations engineer
at a real organization is accountable for — and what this lab
demonstrates.

### Change traceability

Every change must be traceable: who requested it, who approved it, when
it ran, what the result was. The CRQ records in `change-records/` are
this lab's implementation. In production they live in a ticketing system
(ServiceNow, Jira, etc.), but the content is the same.

### Blast radius containment

`serial: 1` in `02-quarterly-update.yml` means the playbook updates one
host at a time. A failure on servera stops the play before touching
serverb, serverc, serverd. In a 200-host environment, you might use `20%`
— the principle is the same: never patch everything at once.

### Rollback before you need it

`04-rollback-checklist.yml` exists not because rollbacks are frequent
but because an operator who has never run a rollback will panic the
first time one is needed. Having a tested, documented procedure reduces
the time-to-recovery and the number of mistakes made under pressure.

### Security by default, not security by afterthought

SELinux is `enforcing` in the role defaults, not `permissive`.
Password SSH authentication is off by default, not "turn it off later."
Audit rules are deployed as part of the baseline, not added after an
incident. The cost of applying a security control to 100 machines with
Ansible is the same as applying it to 1 machine — there is no reason
to defer.

### Idempotency as a reliability guarantee

A playbook you can safely re-run at any time is a playbook you can run
in an emergency without fear of side effects. Idempotency means the
baseline playbook is also the recovery playbook.

---

## How This Maps to RHCE / Linux Admin Skills

| RHCE / exam objective | How this lab covers it |
|-----------------------|------------------------|
| Install and configure an Ansible control node | `ansible.cfg`, `requirements.yml`, `inventory` |
| Configure managed nodes | SSH key-based auth, `ansible_user`, `become` |
| Run and troubleshoot playbooks | All four playbooks; `--check --diff --syntax-check` workflow |
| Create Ansible roles | Five roles with defaults, handlers, templates, tasks |
| Use variables, conditionals, loops, handlers | All roles use all of these |
| Manage users, groups, SSH keys | `users` role |
| Configure SELinux | `baseline_hardening` role, `ansible.posix.selinux` |
| Configure firewalld | `firewall` role, `ansible.posix.firewalld` |
| Configure SSH service | `baseline_hardening` templates and handlers |
| Schedule and manage system updates | `patching` role, `dnf` module |
| Deploy and validate system configuration | `03-validation.yml` with assert tasks |

Beyond exam objectives, this lab also exercises:

- **Change management discipline** — CRQ records, pre-checks, result documentation
- **Audit logging** — auditd rule deployment and verification
- **Time synchronization** — chrony configuration and validation
- **Report generation** — Jinja2 templates producing per-host Markdown reports
- **Rollback planning** — `dnf history undo`, single-host guard, typed confirmation

---

## Future Improvements

These are the things that would make this lab more realistic or
demonstrate additional skills. They are not implemented yet — either
because they would complicate the beginner-friendly structure, or
because they belong in later lab iterations.

- **CRQ-002: Admin user + disable root SSH.** The current lab connects as
  root for convenience. The target state (referenced in
  `change-records/CRQ-002-admin-user-root-ssh-disable.md`) is a dedicated
  `admin` user created by the `users` role, with `PermitRootLogin no`
  enforced by `baseline_hardening`.

- **Inline validation gate in `02-quarterly-update.yml`.** Currently,
  validation is a separate playbook run after patching. A future
  improvement would include the validation tasks as a `post_tasks` block
  inside the update play, making validation a hard gate within the
  `serial: 1` loop.

- **Scheduled drift detection.** A systemd timer or cron job that runs
  `03-validation.yml` weekly would catch configuration drift between
  quarterly patching cycles.

- **Vault-encrypted secrets.** The lab has no secrets today (the break-glass
  account has no password; SSH keys are expected but not stored in the
  repo). A more realistic setup would use `ansible-vault` to encrypt
  sensitive variables and demonstrate vault integration.

- **Lab 2: Prometheus/Grafana monitoring.** The next lab in the portfolio
  adds observability: `node_exporter` on the managed hosts, Prometheus on
  the control node, Grafana for dashboards, and simulated incident
  scenarios to exercise the monitoring stack.

---

---

## CRQ-001 — Q1 Quarterly OE Update (2026-06-06)

### Validation caught a false positive on disk pressure

**What happened:** The first run of `playbooks/03-validation.yml`
failed on all four hosts with:

```
/mnt/repo on /dev/sr0 is 100% full (threshold 70%).
```

`/mnt/repo` is a RHEL ISO mounted as the local DNF repository.
Read-only optical devices always report 100% used — that is
expected behavior, not a problem.

**Why it matters:** A false positive that fires on every host
destroys trust in the validation playbook. If engineers learn to
ignore failures, they will eventually ignore a real one.

**Fix:** Added `-x iso9660` to the `df` exclusion list in
`playbooks/03-validation.yml`. The task name was also updated to
say "writable filesystem" to make the intent clear to future
readers.

**Broader lesson:** Before writing any disk-pressure check, run
`df -T` on the actual target fleet and audit every filesystem type.
Exclude any type that is structurally read-only.

---

### No errata were available this cycle

**What happened:** The quarterly update playbook ran successfully
but reported zero package changes on all hosts. The fleet was
already at the latest available package state.

**Why it matters:** A no-op update run is still a valid run. The
before/after snapshots in `reports/20260606T145955-*.md` are the
evidence — they confirm the fleet was current, not that the
playbook failed.

**For future cycles:** If testing the before/after diff story is
important, install a known-older package before the update run:

```bash
ansible production -m dnf -a 'name=telnet state=present' -b
# run update playbook
ansible production -m dnf -a 'name=telnet state=absent' -b
```

---

### Serial rollout design held under real conditions

**What happened:** `serial: 1` with `max_fail_percentage: 0`
meant each host was updated one at a time, and any failure would
have halted the play before touching subsequent hosts.

**Why it matters:** In the false-positive scenario above, if the
validation assertion had been a hard gate inside the update
playbook, the `serial: 1` design would have stopped after servera
and protected serverb–serverd. That is the intended behavior.

**Current state:** The update and validation playbooks are separate
runs; the validation does not gate the update. A future
improvement (CRQ-002 scope) would be to add an inline
`ansible-playbook playbooks/03-validation.yml --limit {{ inventory_hostname }}`
call after each host's update, making validation a blocking step
within the update loop.

---

## Documentation Discipline

### Change records need real dates from day one

**What happened:** CRQ-001 was committed with placeholder dates
(`YYYY-MM-DD`) and a placeholder owner (`<your name>`). These were
not filled in until after the change was complete.

**Why it matters:** A change record with placeholder content looks
unfinished to a hiring manager and provides no audit value. The
discipline is to fill in the header at the time you schedule the
change, not after.

**For future CRQs:** Fill in Owner, Date submitted, Scheduled
start, and Scheduled end when the record is first created.
Update Actual start/end and Status immediately after the run.

---

### The validation output is evidence — save it explicitly

**What happened:** The validation output was not captured on the
first run attempt (which failed due to the iso9660 false positive).
The clean run was saved to `reports/validation-20260611.txt` and
referenced in CRQ-001.

**Rule:** Every validation run should be tee'd to `reports/`:

```bash
ansible-playbook playbooks/03-validation.yml 2>&1 | \
  tee reports/validation-$(date +%Y%m%d).txt
```

The captured file is the difference between "I ran validation" and
"here is proof that validation passed."
