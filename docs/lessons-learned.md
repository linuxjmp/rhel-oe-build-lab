# Lessons Learned

Accumulated findings from building and operating the
`rhel-oe-build-lab`. Each entry: what happened, why it matters,
and what changed as a result.

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
