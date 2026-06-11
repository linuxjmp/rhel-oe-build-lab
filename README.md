# rhel-oe-build-lab

**Quarterly RHEL Operating Environment Build and Release Pipeline**

A portfolio home lab that simulates how an infrastructure engineer
builds, patches, validates, and rolls back a Red Hat-compatible
Operating Environment. The lab uses Ansible to enforce a baseline,
apply quarterly updates, validate end state, and document every
change the way a real production team would.

---

## Purpose

Demonstrate the full lifecycle of a Linux Operating Environment:

- Baseline hardening
- Quarterly patch deployment
- Post-change validation
- Rollback planning
- Change-record documentation

The lab is designed to read as a production-style workflow rather
than a one-off scripting exercise — every step has an owner, a
validation, and a rollback path.

---

## Architecture

```
+--------------------+
| control-node       |
| Ansible + Git      |
+---------+----------+
          |
          | SSH (key-based) -> sudo -> root
          |
+---------+----+   +---------+   +---------+   +---------+
| servera      |   | serverb |   | serverc |   | serverd |
| App Profile  |   |  App    |   |  App    |   |  App    |
+--------------+   +---------+   +---------+   +---------+
```

See [architecture.md](architecture.md) for the component diagram,
data flow, and security posture.

---

## Skills Demonstrated

- Red Hat Linux administration
- Ansible role and playbook design
- Idempotent configuration management
- Baseline security hardening (SELinux, firewalld, SSH, auditd)
- Patch lifecycle management
- Pre- and post-change validation
- Rollback planning and change-record discipline
- Repository hygiene and infrastructure documentation

---

## Technologies

- RHEL / Rocky Linux / AlmaLinux
- Ansible (`ansible-core`)
- Git
- SSH / OpenSSH
- SELinux, firewalld, auditd, chrony, sudo

---

## Repository Layout

```
rhel-oe-build-lab/
├── README.md
├── architecture.md
├── ansible.cfg
├── inventory
├── group_vars/
├── roles/
│   ├── baseline_hardening/      # scaffolded
│   ├── users/                   # planned
│   ├── firewall/                # planned
│   ├── auditd/                  # planned
│   └── patching/                # planned
├── playbooks/
│   ├── 01-baseline.yml
│   ├── 02-quarterly-update.yml
│   ├── 03-validation.yml
│   ├── 04-rollback-checklist.yml
│   └── templates/
│       └── quarterly_report.md.j2
├── change-records/
│   └── CRQ-001-quarterly-oe-update.md
├── reports/
├── docs/
│   ├── validation-guide.md
│   ├── troubleshooting.md
│   └── lessons-learned.md       # planned
└── screenshots/
```

`# planned` items are intentionally empty so the roadmap is visible.

---

## Setup

On the control node:

```bash
sudo dnf install -y ansible-core git
git clone <this-repo>
cd rhel-oe-build-lab
ansible -i inventory production -m ping
```

Full setup, key distribution, and connectivity verification:
[docs/validation-guide.md](docs/validation-guide.md).

---

## Validation

Standard pre-change pattern (always in this order):

```bash
ansible-playbook -i inventory playbooks/01-baseline.yml --syntax-check
ansible-playbook -i inventory playbooks/01-baseline.yml --check --diff
ansible-playbook -i inventory playbooks/01-baseline.yml
```

The `--check --diff` run produces a diff but makes no changes. It
is the safety net before any apply.

---

## Portfolio Proof

Evidence captured under `screenshots/` and `reports/`:

| File | What it shows |
|------|---------------|
| `screenshots/ansible-ping-success.txt` | Connectivity to all managed hosts |
| `screenshots/validation-output.txt` | Validation playbook — all 13 checks passed, 0 failures |
| `reports/20260606T145955-*.md` | Per-host quarterly update reports (before/after snapshots) |
| `reports/validation-20260611.txt` | Full validation run output with timing |
| `change-records/CRQ-001-quarterly-oe-update.md` | Complete change record with results and lessons learned |

---

## Lessons Learned

[docs/lessons-learned.md](docs/lessons-learned.md) — accumulated
findings from operating this lab: false positives caught, design
decisions validated, documentation discipline notes.

---

## Resume Bullet

> Built a simulated Red Hat Operating Environment release pipeline
> using Ansible, Git, and RHEL-compatible Linux VMs to automate
> baseline hardening, quarterly patch deployment, post-change
> validation, and rollback documentation across four managed hosts.

---

## Interview Explanation

> "I built a small Red Hat operating environment that mirrors the
> quarterly patch cycle a real infrastructure team would run.
> Every step — baseline, patch, validate, document — is automated
> with Ansible and recorded as a change. The point wasn't to write
> a playbook; it was to demonstrate the discipline of running
> changes through pre-checks, validation, and rollback planning."
