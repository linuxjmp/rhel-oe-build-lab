# LinkedIn Project Summary — rhel-oe-build-lab

## One-paragraph summary

I built a RHEL-compatible Operating Environment lab using Ansible to
practice a small infrastructure release lifecycle the way it actually
runs in a managed estate: a security baseline, a quarterly patch
workflow with a post-patch health gate, assertion-based validation,
a guided single-host rollback, and a change record for every change.
Six roles configure SELinux, firewalld, auditd, chrony, sshd, users,
and sudo across four managed hosts; five playbooks orchestrate them.
The point wasn't to automate everything — it was to show controlled
change: syntax-check, dry-run, apply, validate, and keep the evidence.

## Skills demonstrated

- Ansible roles, playbooks, templates, handlers, variables, idempotency
- RHEL administration: SELinux, firewalld, auditd, chrony, sshd, sudo
- Baseline hardening (key-only SSH, root login disabled, audit rules)
- Quarterly patch lifecycle with pre/post snapshots and a health gate
- Validation-driven automation (`assert` with clear PASS/FAIL output)
- Rollback planning and change-control discipline (CRQ records)
- Basic CI: syntax-checks and `ansible-lint` on every push/PR

## What was validated

A real run of `playbooks/03-validation.yml` against all four hosts
passes 21 checks per host with zero failures — SELinux Enforcing,
required services active, SSH password auth disabled, chrony healthy,
firewalld allowing only the expected services, required groups and
sudoers drop-ins present and valid, and the auditd rule set loaded.
The captured output lives in `screenshots/validation-output.txt` and
`reports/validation-20260617.txt`.

## What I'd improve next

- Add an `OnFailure=` alert hook to the weekly drift-detection timer
  (CRQ-005) so a failed check notifies instead of sitting silent.
- Move on to Lab 2: Prometheus/Grafana/node_exporter monitoring with
  simulated incidents, so the validation signal feeds dashboards and
  alerts rather than a one-shot playbook run.
