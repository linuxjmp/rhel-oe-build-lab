# reports/

Per-host markdown reports rendered by
`playbooks/02-quarterly-update.yml`.

## File naming

```
YYYYMMDDTHHMMSS-<hostname>.md
```

Example:

```
20260606T133045-servera.md
20260606T133121-serverb.md
```

The timestamp is the host's `ansible_date_time.iso8601_basic_short`
captured at fact-gather time. With `serial: 1`, each host has its
own timestamp; with higher `serial`, several hosts may share the
same minute — the hostname suffix keeps filenames unique.

## What's in a report

- Kernel before / after
- Whether a reboot was performed (driven by `dnf needs-restarting -r`)
- Baseline service state before / after
- Package diff (added / removed)
- DNF transaction summary

## Workflow

1. Run the quarterly update playbook. Reports land here.
2. Open the relevant change record under `../change-records/` and
   reference the report file(s) in the **Results** section.
3. Commit reports as evidence. Do **not** redact them — the value
   is in the audit trail.

Reports are intentionally **not** gitignored.
